# Data Layout

EDGE uses a customized, fixed format for the storage of dense data (e.g., every element storing degrees of freedom) associated with entities.
The general scheme behind the layout is to emphasize linear access in the computational loops and an "as-is" communication of data.
The term "as-is" means that computational routines and communication routines share the same parts of memory. 

## Entities in Distributed Memory Settings

We distinguish between three types of entities: inner-entities, send-entities and receive-entities.
For element-entities, this usually translates 1-to-1 to the corresponding MPI-functions:
Values of inner-entities are not communicated, those of send-entities are send and values of receive entities are received.

In the setup we derive the inner-, send- and receive-entities in the following way.
First, we consider the owned elements of a partition.
If an element directly shares a face (bridge dimension) with an element owned by a neighboring partition, it is a send-element.
The corresponding element owned by the neighboring partition is a receive-element.
All elements not sharing faces with neighboring partitions are inner-elements.

Now considering faces, those faces who are owned by a partition but don't bridge to an adjacent partition are inner-faces.
Faces owned by a partition bridging to an adjacent partition are send-entities.
Similar faces owned by an adjacent partition bridging to the partition under consideration are receive-faces.
Note that a bridging face is either a send-face or a receive-face, depending on the owning partition, never both.
Also note that only inner-faces and bridging faces exist, this implies that not all faces of ghost-elements are resolved.

For vertices, innner-vertices are those owned by the partition not adjacent to any send-elements.
Send-vertices are owned by the partition and adjacent to send-elements.
Receive-vertices are owned by a neighboring partition and adjacent to receive-elements.
Analogue to the faces, a vertex adjacent to a bridging face is either a send- or receive-vertex, never both.
This choice depends on which partition owns the vertex.
In contrast to the faces, we are also storing vertices adjacent to ghost-entities.

## Local Time Stepping (LTS)

On the outermost level our memory layout we consider the time step of the elements.
Assume given element time step groups with fixed two-fold multiples of a fundamental time step $$\Delta t$$: $$1\cdot \Delta t$$, $$2\cdot \Delta t$$, $$4 \cdot \Delta t$$, etc.
Here, we store (withing one MPI-rank) all elements having time step $$1 \cdot \Delta t$$ first, then all elements having time step $$2 \cdot \Delta t$$, followed by all elements with $$4 \cdot \Delta t$$ and so on.

Now consider a time step group, let's say $$2 \cdot \Delta t$$.
Within a group we store communication-independent inner-elements, with respect to one $$2 \cdot \Delta t$$ time step of this group, first.
Next, we store send-elements, those are elements, which belong to the partition owned by the given MPI-rank and own data, which is required by neighboring partitions.
Last we store, within a time step group, receive-elements.
Receive-elements are owned by neighboring partitions and own data, which is required by the send-elements of the considered partitions.

No particular sorting is performed for inner-elements.

Send-elements are sorted further. Here, we first sort by the time step group of the neighboring LTS-group.
In the case that an send-element has more than one neighboring LTS-group on a remote partition, it is duplicated and stored redundantly in the respective memory regions associated with the respective neighboring LTS-groups.
Now that we have sorted the elements per neighboring LTS-group, we sort the elements by the neighboring MPI-ranks within this group. Once again, if data of a send-element is required by more than one neighboring partition, we duplicate the element as often as required and store it redundantly.
The benefit of duplicating the element logically is that no separate buffers are required for MPI-sends. This means, that we can use the same linear storage of our send-elements for the messages.

The storage of the receive-elements reflects the scheme of the send-elements on the neighboring partition.
Here, we also sort by LTS-group and then by the neighboring rank.
Note that an element from the same rank can exist twice due to the LTS-constraints, since we will need different data in this case.

We follow the same approach for sorting vertices and faces.
In this case, a vertex or face might be adjacent to different time groups.
Thus, we assign the minimum adjacent time group to vertices and faces.

## Global Time Stepping (GTS): A Special Case of LTS

GTS is a special case of LTS with only one time step group having the fundamental time step $$\Delta t$$.
Ignoring the steps above related to different time step groups, the LTS-scheme falls back to a native GTS-algorithm with no LTS-overhead in the data layout.

Since only one time step exists, the outermost level, in terms of GTS, is the splitting into inner-elements, send-elements and receive elements.

The send-elements are sorted by the neighboring MPI-ranks and elements having more than one neighbor are duplicated.

Analogue receive-elements are sorted by the neighboring MPI-ranks.
Since no duplication w.r.t. to time step groups exist, every receive-element is unique per partition.

## Duplicated Entities

The duplication of send-elements leads to non-intuitive situations, one has to be aware of.
It is save to use data of adjacent faces or elements read-only.
However, be aware that writing data of adjacent entities requires special care.

Say, for example (see illustration below), that a mesh-element 00 on partition 0 computes data, which it adds afterwards to the adjacent elements 10 and 20, which reside on partitions 1 and 2 respectively.
Iterating over all send-elements, existing for 00, in our data structure and performing this operations leads to false-behavior:
Here, both redundant copies of element 00, would add data to their adjacent elements.
Further, the inner-element 01 is not aware of the redundant copies, therefore only one of the copies receives the respective update.

```
      ******.************   ...: MPI-boundary
       *01 *  .  20    *    ***: innter-boundary
        * * 00   .   *
         ...........
          *   10  *
            *   *
              *
```

## Sparse Entities
Sparse entities are a small subset of either the vertices, faces or elements in our data layout.
We use sparse entities to encode a special behavior in our solvers.
Examples are faces with (internal) boundary conditions, elements with source terms or elements with receivers.
EDGE assigns every sparse entity an implicit identifier, which follows the storage scheme of the dense data-format:
The first appearing sparse entity of a certain sparse type, e.g. the first face with type "free-surface boundary", has the id 0.
All following entities of this type have ascending ids.
We set the sparse type of an entity by setting a bit in the respective integral type of the entity.
In the integral type bits 0-13 are reserved for mesh-input, e.g. boundary conditions or material layers.
Bit 14 is only true if no mesh-input is present.
Bit 15 is reserved for receivers.
Bits 16-31 are reserved for application-specific parameters.
All remaining bits 32-63 are reserved for the management of time stepping.

Note that having one sparse-type for every dense entity in our dense data layout, means that MPI-duplicates of an entity are not required to have identical sparse types.
For example, first consider, the implementation of point sources in a seismic setup.
Here, all duplicate elements of an element with a source would be marked as entitites with source terms, since we have to apply the sources consistently.
On the contrary, if we would like to extract the solution point-wise through receivers, we only want to do this once.
Therefore, only one of the entities would have a receiver-flag.

## Access of sparse face or vertex data
Sparse faces follow the layout of the dense faces.
As a result linear access to elements and their corresponding sparse faces is, as in the case of dense faces, unstructured in general.
Therefore, if our solver accesses sparse data of faces while iterating over elements, we have to introduce an additional sparse element data structure to cope with this access.

This sparse data structures stores for every element adjacent to one or more sparse faces, the respective sparse offset of all faces.
Faces of such element not having the respective sparse type simply hold an invalid offset.
This scheme is similar to adjacency of elements in the presence of certain boundary conditions, e.g. outflow boundaries.

The same considerations hold for vertex instead of face data.
