# MDDS

MDDS stands for Multi Dimensional Data Structure. It is basically a collection of data structures used to efficiently store as well as query multi-dimensional data for various filtering criteria. It contains the following data structures.

* Segment tree:  a balanced binary tree based data structure. The main feature of this data structure is that it can efficiently detect all intervals (that may overlap) that contain a given point.

* Flat segment tree: a variant of segment tree that handles non-overlapping intervals. This data structure is useful to store values associated with 1-D intervals that never overlap with each other. To know about the features as well as advantages of flat segment tree see [this](http://kohei.us/2010/02/20/increasing-calcs-row-limit-to-1-million/)

* Rectangle set: can store 2-D rectangles. It can be used to efficiently query all rectangles that contain a given point.

* Point quad tree: stores 2-dimensional points and provides an efficient way to query all points within specified rectangular region.

* Mixed type matrix: Stores elements of four types: numeric, string, boolean, and empty. IT can also store flags associated with each element.

* Multi type vector: Can store different unspecified types in a single logical array. Here contiguous elements of identical type are stored in contiguous intervals of memory.

* Multi type matrix: Can store four different types of elements: numeric, string, boolean, and empty.

To know more on mdds try [http://kohei.us/tag/mdds/](http://kohei.us/tag/mdds/)

Let us look at the example of a **flat segment tree**.

It is a templated class having public members like key and value, a structure for non leaf value, a structure for leaf value, some handlers for the template class, iterators, functions for inserting a new segment into the tree, functions for removing a segment from the tree, funtions for shifting the segments left or right, function to perform leaf node search for a value based on a key. The following code was taken from the example code provided along with the mdds source code.
```
typedef ::mdds::flat_segment_tree<long, int> fst_type;

int main() 
{ 
    // Define the begin and end points of the whole segment, and the default value. 
    fst_type db(0, 500, 0); 
    
    db.insert_front(10, 20, 10); 
    db.insert_back(50, 70, 15); 
    db.insert_back(60, 65, 5); 

    int value = -1; 
    long beg = -1, end = -1; 

    // Perform linear search.  This doesn't require the tree to be built beforehand.  Note that the begin and end point parameters are optional. 
    db.search(15, value, &beg, &end); 
    cout << "The value at 15 is " << value << ", and this segment spans from " << beg << " to " << end << endl;; 

    //build the tree first
    db.build_tree(); 

    // Perform tree search.  Tree search is generally a lot faster than linear 
    // search, but requires the tree to be built beforehand. 
    db.search_tree(62, value, &beg, &end); 
    cout << "The value at 62 is " << value << ", and this segment spans from " << beg << " to " << end << endl;; 
}
```