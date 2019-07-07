---
layout:     post
title:      "OpenMesh"
subtitle:   "半边数据结构汇总"
date:       2019-07-06
author:     Mcoder
header-img: img/page_header.jpg
catalog: true
tags:
  - C++
  - openGL
  - Manjaro
  - linux
  - 半边数据结构
---

# 半边数据结构
![](post_img/201901/half_edge.png)
半边数据结构具有如下特点：
1. 每个顶点对应一个出边。例如，一个半边从顶点1出发；
2. 每个面对应一个半边为它的边界，例如面2；
3. 每个半边指向的对象如下：
  - 指向下一个顶点3；
  - 指向一个面4；
  - 指向下一边半边5；
  - 指向反向的半边6；
  - 指向上一个半边7（可选）

# OpenMesh
> OpenMesh uses a lazy deletion scheme to avoid unnecessary updates to the data structure. The halfedge data structure will always be updated directly to ensure that following algorithms will have the correct iterator setups.
>
> So if you delete a face, The face itself will still exist but the halfedges which are now located at the hole will be updated directly, which means that circulators on the adjacent vertices will not come across the face anymore.
>
> If an edge is deleted, the adjacent faces will be removed as well (flagging them deleted and updating the surrounding halfedges). The edge itself will also be flagged as deleted. Again the circulators will not see the deleted primitives anymore.
>
> For a vertex, all adjacent faces and edges are deleted with the schemes above and the vertex flagged as deleted.
>
> The iterators, going across vertices edges and faces will still enumerate all primitives (including deleted ones). Except if you use the skipping iterators, which will skip deleted primitives. The circulators always only enumerate primitives which are not deleted.
# 想关算法

## OpenMesh 的迭代器

### 直接遍历所有的点、半边、边、面
```C++
MyMesh mesh;
// iterate over all vertices
for (MyMesh::VertexIter v_it=mesh.vertices_begin(); v_it!=mesh.vertices_end(); ++v_it) 
   ...; // do something with *v_it, v_it->, or *v_it
// iterate over all halfedges
for (MyMesh::HalfedgeIter h_it=mesh.halfedges_begin(); h_it!=mesh.halfedges_end(); ++h_it) 
   ...; // do something with *h_it, h_it->, or *h_it
// iterate over all edges
for (MyMesh::EdgeIter e_it=mesh.edges_begin(); e_it!=mesh.edges_end(); ++e_it) 
   ...; // do something with *e_it, e_it->, or *e_it
// iterator over all faces
for (MyMesh::FaceIter f_it=mesh.faces_begin(); f_it!=mesh.faces_end(); ++f_it) 
   ...; // do something with *f_it, f_it->, or *f_it
```

因为 OpenMesh 是一个比较”懒惰“的模式，即当做出少量修改时，他不会真的修改底层数据结构，而是通过标记确定哪些顶点、边、面已经失效了。而上方的迭代器会遍历所有，包括已经失效的。`Skipping Iterators`可以解决这个问题。`vertices_sbegin(), edges_sbegin(),halfedges_sbegin(),faces_sbegin()`。

### 局部的迭代器 `Circulators`

关于点的循环:

- VertexVertexIter: iterate over all neighboring vertices.
- VertexIHalfedgeIter: iterate over all incoming halfedges.
- VertexOHalfedgeIter: iterate over all outgoing halfedges.
- VertexEdgeIter: iterate over all incident edges.
- VertexFaceIter: iterate over all adjacent faces.

关于面的循环：

- FaceVertexIter: iterate over the face's vertices.
- FaceHalfedgeIter: iterate over the face's halfedges.
- FaceEdgeIter: iterate over the face's edges.
- FaceFaceIter: iterate over all edge-neighboring faces.

其他：

- HalfedgeLoopIter: iterate over a sequence of Halfedges. (all Halfedges over a face or a hole)

```C++
/**************************************************
 * Vertex circulators
 **************************************************/
// Get the vertex-vertex circulator (1-ring) of vertex _vh
VertexVertexIter OpenMesh::PolyConnectivity::vv_iter (VertexHandle _vh);
// Get the vertex-incoming halfedges circulator of vertex _vh
VertexIHalfedgeIter OpenMesh::PolyConnectivity::vih_iter (VertexHandle _vh);
// Get the vertex-outgoing halfedges circulator of vertex _vh
VertexOHalfedgeIter OpenMesh::PolyConnectivity::voh_iter (VertexHandle _vh);
// Get the vertex-edge circulator of vertex _vh
VertexEdgeIter OpenMesh::PolyConnectivity::ve_iter (VertexHandle _vh);
// Get the vertex-face circulator of vertex _vh
VertexFaceIter OpenMesh::PolyConnectivity::vf_iter (VertexHandle _vh);
/**************************************************
 * Face circulators
 **************************************************/
// Get the face-vertex circulator of face _fh
FaceVertexIter OpenMesh::PolyConnectivity::fv_iter (FaceHandle _fh);
// Get the face-halfedge circulator of face _fh
FaceHalfedgeIter OpenMesh::PolyConnectivity::fh_iter (FaceHandle _fh);
// Get the face-edge circulator of face _fh
FaceEdgeIter OpenMesh::PolyConnectivity::fe_iter (FaceHandle _fh);
// Get the face-face circulator of face _fh
FaceFaceIter OpenMesh::PolyConnectivity::ff_iter (FaceHandle _fh);
```

### 循环每一个顶点的邻居节点
```C++
MyMesh mesh;
// (linearly) iterate over all vertices
for (MyMesh::VertexIter v_it=mesh.vertices_sbegin(); v_it!=mesh.vertices_end(); ++v_it)
{
  // circulate around the current vertex
  for (MyMesh::VertexVertexIter vv_it=mesh.vv_iter(*v_it); vv_it.is_valid(); ++vv_it)
  {
    // do something with e.g. mesh.point(*vv_it)
  }
}
```

### 循环每一个面的半边
```C++
MyMesh mesh;
...
// Assuming faceHandle contains the face handle of the target face
MyMesh::FaceHalfedgeIter fh_it = mesh.fh_iter(faceHandle);
for(; fh_it.is_valid(); ++fh_it) {
    std::cout << "Halfedge has handle " << *fh_it << std::endl;
}
```

### 通过半边的`next_halfedge`一直遍历下去
```C++
[...]
TriMesh::HalfedgeHandle heh, heh_init;
// Get the halfedge handle assigned to vertex[0]
heh = heh_init = mesh.halfedge_handle(vertex[0].handle());
// heh now holds the handle to the initial halfedge.
// We now get further on the boundary by requesting
// the next halfedge adjacent to the vertex heh
// points to...
heh = mesh.next_halfedge_handle(heh);
// We can do this as often as we want:
while(heh != heh_init) {
        heh = mesh.next_halfedge_handle(heh);
}
[...]
```

### 关于网格模型的边界
```C++
// Test if a halfedge lies at a boundary (is not adjacent to a face)
bool is_boundary (HalfedgeHandle _heh) const
// Test if an edge lies at a boundary
bool is_boundary (EdgeHandle _eh) const
// Test if a vertex is adjacent to a boundary edge
bool is_boundary (VertexHandle _vh) const
// Test if a face has at least one adjacent boundary edge.
// If _check_vertex=true, this function also tests if at least one
// of the adjacent vertices is a boundary vertex
bool is_boundary (FaceHandle _fh, bool _check_vertex=false) const

```

### 找到一个顶点所有的出半边、入半边

```C++
[...]
// Get some vertex handle
PolyMesh::VertexHandle v = ...;
for(PolyMesh::VertexIHalfedgeIter vih_it = mesh.vih_iter(v); vih_it; ++vih_it) {
        // Iterate over all incoming halfedges...
}
for(PolyMesh::VertexOHalfedgeIter voh_it = mesh.voh_iter(v); voh_it; ++voh_it) {
        // Iterate over all outgoing halfedges...
}
[...]
```

### 找到一组半边中的另一个

```C++
// Get the halfedge handle of i.e. the halfedge
// that is associated to the first vertex
// of our set of vertices
PolyMesh::HalfedgeHandle heh = mesh.halfedge_handle(*(mesh.vertices_begin()));
// Now get the handle of its opposing halfedge
PolyMesh::HalfedgeHandle opposite_heh = mesh.opposite_halfedge_handle(heh);

// Get the face adjacent to the opposite halfedge
OpenMesh::PolyConnectivity::opposite_face_handle();
// Get the handle to the opposite halfedge
OpenMesh::Concepts::KernelT::opposite_halfedge_handle();
// Get the opposite vertex to the opposite halfedge
OpenMesh::TriConnectivity::opposite_he_opposite_vh();
// Get the vertex assigned to the opposite halfedge
OpenMesh::TriConnectivity::opposite_vh();

```

# Reference

1. [OpenMesh](https://www.openmesh.org/media/Documentations/OpenMesh-Doc-Latest/a04074.html)
2. [OpenMesh-Mesh Iterators and Circulators](https://www.openmesh.org/media/Documentations/OpenMesh-Doc-Latest/a04080.html)
3. [OpenMesh-How to navigate on a mesh](https://www.openmesh.org/media/Documentations/OpenMesh-Doc-Latest/a04082.html)