---
layout: post
title: "Self Balancing Binary Search Tree(2) - Red-Black Tree"
categories: [all, datastructures]
date: 2015-04-16
author: Rui Zhang
---

The red-black tree is invented by Rudolf Bayer in 1972. It's one of the most famous self-balancing binary search tree. TreeMap/TreeSet in Java and std::map/std::set in C++ STL use red-black tree as underlying data structure. Other popular self-balancing search trees include 2-3 tree, AA tree, AVL tree, Splay tree and Treap[^1].

### Property
Red-Black tree has 5 properties[^2].   
1. Every node is either red or black  
2. The root is black  
3. Every leaf is black  
4. There is no two adjacent red nodes  
5. For each node, all simple paths from the node to a leaf contain the same number of black nodes.  

A red-black tree:  
![fig1]  

The property above guarantees the height of a red-black tree is at most 2log(n+1). Therefore red-black tree guarantees common operations in O(log n) time.

|Space|O(n)|
|Search|O(log n)|
|Insert|O(log n)|
|Delete|O(log n)|
|Successor|O(log n)<sup>[1]</sup>|
|Predecessor|O(log n)<sup>[1]</sup>|
{: class="table table-striped table-nonfluid"}
[1]. without extra pointer

### Node Structure
A red-black tree node stores one extra field color for balancing. 
{% highlight c++ %}
enum class Color {RED, BLACK};

template <typename T, typename U>
struct RBTreeNode {
    T key;
    U value;
    Color color;
    RBTreeNode *parent;
    RBTreeNode *left;
    RBTreeNode *right;
    RBTreeNode(T k, U v) : key(k), value(v), color(Color::RED),
                           parent(nullptr), left(nullptr), right(nullptr) { }
    ~RBTreeNode() {
        delete left;
        delete right;
    }
};

<!-- An extra sentinel Nil is used instead of nullptr to simplify the implementation. -->

template <typename T, typename U>
class RBTree {
private:
    RBTreeNode<T, U> *root_;
    int size_;

    RBTreeNode<T, U>* GetUncle(RBTreeNode<T, U> *x);

    void LeftRotation(RBTreeNode<T, U> *x);

    void RightRotation(RBTreeNode<T, U> *x);

    void RBInsertFixup(RBTreeNode<T, U> *x);

    void RBDelete(RBTreeNode<T, U> *z);

    void RBDeleteFixup(RBTreeNode<T, U> *x);

public:
    RBTree() : root_(nullptr), size_(0) { }

    ~RBTree() { delete root_; }

    int size() const { return size_; }

    RBTreeNode<T, U>* Search(T key);

    void RBInsert(T key, U value);  

    void RBDelete(T key);

    RBTreeNode<T, U>* Successor(RBTreeNode<T, U> *z);

    RBTreeNode<T, U>* Predecessor(RBTreeNode<T, U> *z);
};
{% endhighlight %}

### Search, Successor, Predecessor
Search, Successor and Predecessor of red-black tree are the same as a normal BST.
{% highlight c++ %}
template <typename T, typename U>
RBTreeNode<T, U>* RBTree<T, U>::Search(T key) {
    RBTreeNode<T, U> *x = root_;
    while (x) {
        if (key == x->key) { 
            return x;
        } else if (key < x->key) {
            x = x->left;
        } else {
            x = x->right;
        }
    }
    return nullptr;
}

template <typename T, typename U>
RBTreeNode<T, U>* Successor(RBTreeNode<T, U> *z) {
    RBTreeNode<T, U> *x = z->right;
    if (x) {
        while (x->left) {
            x = x->left;
        }
        return x;
    } else {
        RBTreeNode<T, U> *y = z->parent;
        while (y && z == y->right) {
            z = y;
            y = y->parent;
        }
        return y;
    }
}

template <typename T, typename U>
RBTreeNode<T, U>* Predecessor(RBTreeNode<T, U> *z) {
    RBTreeNode<T, U> *x = z->left;
    if (x) {
        while (x->right) {
            x = x->right;
        }
        return x;
    } else {
        RBTreeNode<T, U> *y = z->parent;
        while (y && z == y->left) {
            z = y;
            y = y->parent;
        }
        return y;
    }
}

{% endhighlight %}

### Insertion[^3]
The newly inserted node is always set to red. Similar to other self-balancing trees, to guarantee the red-black tree properties are preserved, auxiliary fix-up is needed after insertion and deletion. There are two kinds of fix-up for red-black tree: **Recoloring** and **Rotation**.

{% highlight c++ %}
template <typename T, typename U>
void RBTree<T, U>::RBInsert(T key, U value) {
    RBTreeNode<T, U> *y = nullptr;
    RBTreeNode<T, U> *x = root_;
    while (x) {
        y = x;
        if (key == x->key) { // if the key exists, update value
            x->value = value;
            return;
        } else if (key < x->key) {
            x = x->left;
        } else {
            x = x->right;
        }
    }
    RBTreeNode<T, U> *z = new RBTreeNode<T, U>(key, value);
    z->parent = y;
    if (!y) {
        root_ = z;
    } else if (key < y->key) {
        y->left = z;
    } else {
        y->right = z;
    }
    RBInsertFixup(z);
}
{% endhighlight %}
There are 6 cases in red-black tree insertion fix-up differ in uncle color or subtree structure.

Denote the newly inserted node as *x*.

**Case 1**: *z* is root.
Fix-up: Only need to change x's color to black
{% highlight c++ %}
if (!x->parent) {
    x->color = Color::BLACK;
}    
{% endhighlight %}
-------------------------------------------------------

In case 2-6 we need to know the color of the uncle node. The function below return the uncle of x. Note that null node is considered black in red-black tree.  
{% highlight c++ %}
template <typename T, typename U>
RBTreeNode<T, U>* RBTree<T, U>::GetUncle(RBTreeNode<T, U> *x) {
    RBTreeNode<T, U>* p = x->parent;
    RBTreeNode<T, U>* g = x->parent->parent;
    if (!g) {
        return nullptr;
    }     
    return p == g->left ? g->right : g->left; 
}
{% endhighlight %}
-------------------------------------------------------

**Case 2**: x's uncle is Red.   
![fig2]  
**Fix-up**:  
(1) set color of parent and uncle to black  
(2) change color of grandparent to red  
(3) reassign x = x's grandparent and run fix-up for new x.  

![fig3]  

**Implementation**: 
{% highlight c++ %}
if (u && u->color == Color::RED) {
    p->color = Color::BLACK;
    u->color = Color::BLACK;
    g->color = Color::RED;
    x = g;
}
{% endhighlight %}

-------------------------------------------------------

**Case 3**: x's uncle is Black, parent is left child of grandparent and x is left child of parent(LL).  
![fig4]  
**Fix-up**:  
(1) swap color of g and p  
(2) right rotation on g 

![fig5]  

**Implementation**: 
{% highlight c++ %}
if ((!u || u->color == Color::BLACK) && x == p->left && p == g->left) { 
    p->color = Color::BLACK;
    g->color = Color::RED;
    RightRotation(g);
}
{% endhighlight %}

-------------------------------------------------------

**Case 4**: x's uncle is Black, parent is left child of grandparent and x is right child of parent(LR).  
![fig6]  
**Fix-up**:  
(1) left rotation on p  
(2) converted to LL case  

![fig7]  

**Implementation**: 
{% highlight c++ %}
if ((!u || u->color == Color::BLACK) && x == p->right && p == g->left) { 
    LeftRotation(p);
    x->color = Color::BLACK;
    g->color = Color::RED;
    RightRotation(g);
}
{% endhighlight %}
-------------------------------------------------------

**Case 5**: x's uncle is Black, parent is right child of grandparent and x is right child of parent(RR).  
![fig8]  
**Fix-up**:  
(1) swap color of g and p  
(2) left rotation on g  

![fig9]  

**Implementation**: 
{% highlight c++ %}
if ((!u || u->color == Color::BLACK) && x == p->right && p == g->right) { 
    p->color = Color::BLACK;
    g->color = Color::RED;
    LeftRotation(g);
}
{% endhighlight %}
-------------------------------------------------------

**Case 6**: x's uncle is Black, parent is right child of grandparent and x is left child of parent(RL).  
![fig10]  
**Fix-up**:  
(1) right rotation on p  
(2) converted to RR case  

![fig11]  

**Implementation**: 
{% highlight c++ %}
if ((!u || u->color == Color::BLACK) && x == p->left && p == g->right) { 
    RightRotation(p);
    x->color = Color::BLACK;
    g->color = Color::RED;
    LeftRotation(g);
}
{% endhighlight %}
-------------------------------------------------------

#### **Complete Insertion Fix-up**

{% highlight c++ %}
template <typename T, typename U>
void RBTree<T, U>::RBInsertFixup(RBTreeNode<T, U> *x) {
    while (x != root_ && x->parent->color == Color::RED) { // x is red at the beginning of each iteration
        RBTreeNode<T, U>* u = GetUncle(x);
        RBTreeNode<T, U>* p = x->parent;
        RBTreeNode<T, U>* g = p->parent;   // g can't be NULL since x->parent->color == Color::RED
        if (u && u->color == Color::RED) { // uncle is red
            u->color = Color::BLACK;
            x = g;
        } else if (p == g->left) {                 
            if (x == p->right) {             // LR case                
                LeftRotation(p);
                std::swap(x, p);
            }
            RightRotation(g);                // LL case
        } else { 
            if (x == p->left) {              // RL case                
                RightRotation(p);
                std::swap(x, p);
            }
            LeftRotation(g);                 // RR case
        }
        p->color = Color::BLACK;         
        g->color = Color::RED; 
    }
    root_->color = Color::BLACK;
}
{% endhighlight %}

### Deletion
Deletion is more complex than Insertion. It consists of a modified binary search tree deletion and a red-black property violation fix-up.

There are 3 cases when deleting a node x in standard binary search tree.[^2] Let p = x->parent. 

(1) x has not child, modify p to replace x with NULL.

![fig12]

(2) x has one child, find a suitable child y in subtree rooted at x and modify p to replace x with y.

![fig13]

(3) x has two child, find x's successor y and replace x with s.

* if y is x's right child, replace x with y directly

![fig14]

* if y is not x's right child, replace y with it's own right child, then replace x with y.

![fig15]

The function below delete node z in binary search tree.
{% highlight c++ %}
void BSTDelete(TreeNode *z) {
    if (!z->left) {
        Transplant(z, z->right);
    } else if (!z->right) {
        Transplant(z, z->left);
    } else {
        y = z->right;
        while (y->left) {
            y = y->left;
        }
        if (y->parent != z) {
            Transplant(y, y->right);
            y->right = z->right;
            y->right->parent = y;
        }
        Transplant(z, y);
        y->left = z->left;
        y->left->parent = y;
    }
}

// replace the subtree root at u with subtree rooted at v
void Transplant(TreeNode *u, TreeNode *v) {
    TreeNode *p = u->parent;
    if (!p) {
        root_ = v;
    } else if (u == p.left) {
        p.left = v;
    } else {
        p.right = v;
    }
    if (!v) {
        v->parent = parent;
    }
}
{% endhighlight %}
<br>
The **RBDelete** function add additional code to keep track with the node which may cause red-black property violation.

{% highlight c++ %}
template <typename T, typename U>
void RBTree<T, U>::RBDelete(RBTreeNode<T, U> *z) {
    RBTreeNode<T, U> *x = nullptr;
    RBTreeNode<T, U> *y = z;
    Color y_original_color = y->color;
    if (!z->left) {
        x = z->right;
        Transplant(z, z->right);
    } else if (!z->right) {
        x = z->left;
        Transplant(z, z->left);
    } else {
        y = z->right;
        while (y->left) {
            y = y->left;
        }
        y_original_color = y->color;
        x = y->right;
        if (y->parent == z) {
            x->parent = y;
        } else {
            Transplant(y, y->right);
            y->right = z->right;
            y->right->parent = y;
        }
        Transplant(z, y);
        y->left = z->left;
        y->left->parent = y;
        y->color = z->color;
    }
    delete z;
    if (y_original_color == Color::BLACK) {
        RBDeleteFixup(x);
    }
}

template <typename T, typename U>
void RBTree<T, U>::RBDelete(T key) {
    RBDelete(Search(key));
}
{% endhighlight %}
There are 8 cases in red-black tree deletion fix-up. The first four and the last four are symmetric.

**Case 1**: x's sibling w is red.

![fig16]  
**Fix-up**:  
(1) switch color of w and p  
(2) left rotation on p
(3) now x's sibling is a black node, case 1 is converted to case 2, 3 or 4

-------------------------------------------------------

**Case 2**: x's sibling w is black, and both of w's children are black.  

![fig17]  
**Fix-up**:  
(1) set color of w to red    
(2) left rotation on p  
(3) reassign p to x, and continue fix-up on new x

-------------------------------------------------------

**Case 3**: x's sibling w is black, w's left child is red, w's right child is black
![fig18]  
**Fix-up**:  
(1) switch color of w and w's left child  
(2) right rotation w  
(3) now sibling of x has a red right child, case 3 is converted to case 4

-------------------------------------------------------

**Case 4**: x's sibling w is black, and w's right child is red

(1)  w's left child is black  
![fig19]  

(2)  w's left child is red  
![fig20]  

**Fix-up**:  
 (1) switch color of p and w  
 (2) set w's right child to red  
 (3) left rotation on p

-------------------------------------------------------

**Case 5-8** are symmetric, same as Case 1-4 with "right" and "left" exchanged 

-------------------------------------------------------

**Implementation**:  

{% highlight c++ %}
template <typename T, typename U>
void RBTree<T, U>::RBDeleteFixup(RBTreeNode<T, U> *x) {
    while (x != root_ && x->color == Color::BLACK) {
        RBTreeNode<T, U> *p = x->parent;
        if (x == p->left) {
            RBTreeNode<T, U> *w = p->right;
            if (w->color == Color::RED) {            
                w->color = Color::BLACK;
                p->color = Color::RED;
                LeftRotation(p);
                w = p->right;
            }
            if (w->left->color == Color::BLACK && w->right->color == Color::BLACK) {
                w->color = Color::RED;
                x = p;
            } else {
                if (w->right->color == Color::BLACK) {
                    w->left->color = Color::BLACK;
                    w->color = Color::RED;
                    RightRotation(w);
                    w = p->right;
                } 
                w->color = p->color;
                p->color = Color::BLACK;
                w->right->color = Color::BLACK;
                LeftRotation(p);
                x = root_;
            }
        } else {
            RBTreeNode<T, U> *w = p->left;
            if (w->color == Color::RED) {
                w->color = Color::BLACK;
                p->color = Color::RED;
                RightRotation(p);
                w = p->left;
            }
            if (w->left->color == Color::BLACK && w->right->color == Color::BLACK) {
                w->color = Color::RED;
                x = p;
            } else {
                if (w->left->color == Color::BLACK) {
                    w->right->color = Color::BLACK;
                    w->color = Color::RED;
                    LeftRotation(w);
                    w = p->left;
                } 
                w->color = p->color;
                p->color = Color::BLACK;
                w->left->color = Color::BLACK;
                RightRotation(p);
                x = root_;
            }
        }
    }
    x->color = Color::BLACK;
}
{% endhighlight %}

**Reference:**

[^1]: [Wikipedia: Self-balancing binary search tree ](https://en.wikipedia.org/wiki/Self-balancing_binary_search_tree)

[^2]: [Introduction to Algorithms, 3rd Edition ](http://www.amazon.com/Introduction-Algorithms-Edition-Thomas-Cormen/dp/0262033844)

[^3]: [Geeksforgeeks: Red-Black Tree Set 2 (Insert)](http://www.geeksforgeeks.org/red-black-tree-set-2-insert)

[fig1]: /assets/RBTree/fig1.png

[fig2]: /assets/RBTree/fig2.png

[fig3]: /assets/RBTree/fig3.png

[fig4]: /assets/RBTree/fig4.png

[fig5]: /assets/RBTree/fig5.png

[fig6]: /assets/RBTree/fig6.png

[fig7]: /assets/RBTree/fig7.png

[fig8]: /assets/RBTree/fig8.png

[fig9]: /assets/RBTree/fig9.png

[fig10]: /assets/RBTree/fig10.png

[fig11]: /assets/RBTree/fig11.png

[fig12]: /assets/RBTree/fig12.png

[fig13]: /assets/RBTree/fig13.png

[fig14]: /assets/RBTree/fig14.png

[fig15]: /assets/RBTree/fig15.png

[fig16]: /assets/RBTree/fig16.png

[fig17]: /assets/RBTree/fig17.png

[fig18]: /assets/RBTree/fig18.png

[fig19]: /assets/RBTree/fig19.png

[fig20]: /assets/RBTree/fig20.png