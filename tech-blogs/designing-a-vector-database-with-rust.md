# Designing a Vector Database with Rust

#### **Introduction**

In the age of machine learning and big data, vectors have become a ubiquitous way to represent complex data structures. Whether it's for word embeddings, image features, or user profiles, the ability to efficiently store and retrieve vectors is crucial. Enter VectorDb, a database designed specifically for vectors. And what better language to implement it in than Rust, known for its emphasis on safety, concurrency, and performance?

#### **Why VectorDb?**

Traditional databases are optimized for structured data like rows and columns. However, vectors, especially in high-dimensional spaces, require specialized storage and retrieval mechanisms. A vector database like VectorDb provides:

1. **Efficient Storage**: Compact representation of vectors.
2. **Fast Retrieval**: Quick similarity searches in high-dimensional spaces.
3. **Scalability**: Handling millions to billions of vectors.

#### **Why Rust?**

Rust is not just another programming language; it's a revolution in systems programming:

* **Memory Safety**: Rust's ownership system ensures no null pointer dereferences, data races, or other common bugs.
* **Concurrency**: Built-in concurrency primitives for building scalable systems.
* **Performance**: Zero-cost abstractions ensure high-speed operations.

#### **Designing VectorDb**

**1. Data Structures**:

* **Vector**: A struct containing an ID and a vector.
* **Index**: A structure for efficient search mechanisms.

**2. Storage**: A flat file might seem primitive, but with the right serialization techniques, it can be incredibly efficient. Each vector is serialized and appended, ensuring quick writes.

**3. Searching**: The heart of VectorDb. We'll use the Annoy (Approximate Nearest Neighbors Oh Yeah) algorithm, a popular choice for high-dimensional vector searches.



### What's the underlying data structure for ANNOY

Suppose we have all our vectors in a space, we can break up the space into hyperplanes like this

<figure><img src="../.gitbook/assets/Screenshot 2023-12-02 at 11.32.04 PM.png" alt=""><figcaption><p>(Image taken from <a href="https://erikbern.com/2015/10/01/nearest-neighbors-and-vector-models-part-2-how-to-search-in-high-dimensional-spaces.html">https://erikbern.com/2015/10/01/nearest-neighbors-and-vector-models-part-2-how-to-search-in-high-dimensional-spaces.html</a></p></figcaption></figure>

These hyperplanes are represented as binary trees in our code. We'll build the tree when we build the index, and to find the k Nearest neighbor, we'll just traverse the tree to find the lead node that the query point is in, and return the vectors that're in that node.&#x20;

<figure><img src="../.gitbook/assets/Screenshot 2023-12-02 at 11.35.51 PM.png" alt=""><figcaption><p>Image taken from: <a href="https://erikbern.com/2015/10/01/nearest-neighbors-and-vector-models-part-2-how-to-search-in-high-dimensional-spaces.html">https://erikbern.com/2015/10/01/nearest-neighbors-and-vector-models-part-2-how-to-search-in-high-dimensional-spaces.html</a></p></figcaption></figure>

This works as a starting point, but it could result in not including some neighbors that are close, but is split into a a different hyperple. We can improve this by building multiple trees to improve the possibility of covering a vector's close neighbors

### **Implementation**&#x20;

**For Constructing the index by building the trees.**

```rust

/// Builds a hyperplane from a set of indexes and vectors.
/// 
/// # Arguments
/// 
/// * `indexes` - A reference to a vector of usize indexes.
/// * `all_vectors` - A reference to a vector of vectors.
/// 
/// # Returns
/// 
/// A tuple containing the created HyperPlane and two vectors of usize,
/// representing the indexes above and below the hyperplane.
fn construct_hyperplane<const N: usize>(
    indexes: &[usize],
    all_vectors: &[Vector<N>],
) -> (HyperPlane<N>, Vec<usize>, Vec<usize>) {
    // Randomly choose two distinct indexes.
    let mut rng = rand::thread_rng();
    let selected_indexes = indexes
        .choose_multiple(&mut rng, 2)
        .copied()
        .collect::<Vec<_>>();
    let (index_a, index_b) = (selected_indexes[0], selected_indexes[1]);

    // Calculate the coefficients and constant for the hyperplane equation.
    let coefficients = all_vectors[index_a].subtract_from(&all_vectors[index_b]);
    let mid_point = all_vectors[index_a].avg(&all_vectors[index_b]);
    let constant = -coefficients.dot_product(&mid_point);

    // Create the hyperplane.
    let hyperplane = HyperPlane {
        coefficients,
        constant,
    };

    // Divide indexes into two groups based on their position relative to the hyperplane.
    let (above_indexes, below_indexes) = indexes.iter()
        .partition(|&&idx| hyperplane.point_is_above(&all_vectors[idx]));

    (hyperplane, above_indexes, below_indexes)
}

/// Builds a tree for Approximate Nearest Neighbor search.
/// 
/// # Arguments
/// 
/// * `max_size` - The maximum size of leaf nodes in the tree.
/// * `indexes` - A reference to a vector of usize indexes.
/// * `all_vectors` - A reference to a vector of vectors.
/// 
/// # Returns
/// 
/// A Node representing either a leaf or an inner node of the tree.
fn construct_tree<const N: usize>(
    max_size: i32,
    indexes: &[usize],
    all_vectors: &[Vector<N>],
) -> Node<N> {
    // If the number of indexes is within the max size, create a leaf node.
    if indexes.len() <= max_size as usize {
        Node::Leaf(Box::new(LeafNode(indexes.to_vec())))
    } else {
        // Otherwise, split the indexes and recursively construct child nodes.
        let (hyperplane, above_indexes, below_indexes) = construct_hyperplane(indexes, all_vectors);
        let node_above = construct_tree(max_size, &above_indexes, all_vectors);
        let node_below = construct_tree(max_size, &below_indexes, all_vectors);

        Node::Inner(Box::new(InnerNode {
            hyperplane,
            left_node: node_below,
            right_node: node_above,
        }))
    }
}

/// Constructs an Approximate Nearest Neighbor Index.
/// 
/// # Arguments
/// 
/// * `num_trees` - The number of trees to construct in the index.
/// * `max_size` - The maximum size of leaf nodes in the tree.
/// * `vectors` - A reference to a vector of vectors.
/// * `vector_ids` - A reference to a vector of vector IDs.
/// 
/// # Returns
/// 
/// An ANNIndex containing the constructed trees and vector information.
pub fn construct_index<const N: usize>(
    num_trees: i32,
    max_size: i32,
    vectors: &[Vector<N>],
    vector_ids: &[i32],
) -> ANNIndex<N> {
    let (mut deduped_vectors, mut deduped_ids) = (vec![], vec![]);
    Self::deduplicate(vectors, vector_ids, &mut deduped_vectors, &mut deduped_ids);

    // Create indexes for all unique vectors.
    let all_indexes: Vec<usize> = (0..deduped_vectors.len()).collect();

    // Build trees in parallel.
    let trees: Vec<_> = (0..num_trees)
        .into_par_iter()
        .map(|_| construct_tree(max_size, &all_indexes, &deduped_vectors))
        .collect();

    ANNIndex::<N> {
        trees,
        ids: deduped_ids,
        values: deduped_vectors,
    }
}

```

#### For querying a vector's nearest neighbors:

```rust
/// Searches for candidates in a tree and adds them to a concurrent hash set.
///
/// # Arguments
///
/// * `query_vector` - The vector to be queried.
/// * `num_candidates` - The number of candidates to find.
/// * `current_node` - A reference to the current node in the tree.
/// * `candidate_set` - A concurrent hash set to store the candidate indexes.
///
/// # Returns
///
/// The number of candidates found in this tree.
fn search_candidates<const N: usize>(
    query_vector: Vector<N>,
    num_candidates: i32,
    current_node: &Node<N>,
    candidate_set: &DashSet<usize>,
) -> i32 {
    match current_node {
        Node::Leaf(leaf) => {
            let leaf_indexes = &leaf.0;
            let found_candidates = std::cmp::min(num_candidates as usize, leaf_indexes.len());
            for &index in leaf_indexes.iter().take(found_candidates) {
                candidate_set.insert(index);
            }
            found_candidates as i32
        }
        Node::Inner(inner_node) => {
            let is_above = inner_node.hyperplane.point_is_above(&query_vector);
            let (primary_branch, alternate_branch) = if is_above {
                (&inner_node.right_node, &inner_node.left_node)
            } else {
                (&inner_node.left_node, &inner_node.right_node)
            };

            match search_candidates(query_vector, num_candidates, primary_branch, candidate_set) {
                found if found < num_candidates => {
                    found + search_candidates(query_vector, num_candidates - found, alternate_branch, candidate_set)
                }
                found => found,
            }
        }
    }
}

/// Performs an approximate search for the nearest neighbors of a query vector.
///
/// # Arguments
///
/// * `query_vector` - The query vector.
/// * `num_top_candidates` - The number of nearest neighbors to find.
///
/// # Returns
///
/// A vector of tuples containing the index and the squared Euclidean distance
/// of each of the nearest neighbors.
pub fn search_approximate<const N: usize>(
    &self,
    query_vector: Vector<N>,
    num_top_candidates: i32,
) -> Vec<(i32, f32)> {
    let candidate_set = DashSet::new();
    self.trees.par_iter().for_each(|tree| {
        search_candidates(query_vector, num_top_candidates, tree, &candidate_set);
    });

    candidate_set
        .into_iter()
        .filter_map(|idx| {
            self.values.get(idx).map(|vector| {
                let distance = vector.sq_euc_dis(&query_vector);
                (self.ids[idx], distance)
            })
        })
        .sorted_by(|a, b| a.1.partial_cmp(&b.1).unwrap())
        .take(num_top_candidates as usize)
        .collect()
}

```

####

#### **Challenges and Solutions**

1. **Memory Usage**: High-dimensional vectors can be memory-intensive. Solution? Memory-mapped files to reduce RAM usage.
2. **Concurrency**: Multiple reads and writes can cause data races. Rust's Arc and Mutex ensure thread-safe access.
3. **Scaling**: Large datasets can be daunting. Sharding data across multiple VectorDb instances can be a game-changer.

#### **Future Enhancements**

* **Distributed Search**: A distributed VectorDb to scale across machines.
* **Compression**: Techniques like quantization to reduce storage needs.

#### **Conclusion**

Building a vector database in Rust is a journey filled with challenges and learning. But with Rust's features and the right design choices, VectorDb can be a powerful tool in the modern data ecosystem.
