---
title: 二叉树的前中后序遍历的递归与非递归实现
key: 20180826
tags:
- Binary Tree Traversal
- Stack
- Recursion
---

## 二叉树的递归遍历 ##
<!--more-->

    class Solution {
		public:
	    vector<int> preorderTraversal(TreeNode *root) {
	        if (root) {
	            m_result.push_back(root->val);
	            preorderTraversal(root->left);
	            preorderTraversal(root->right);
	        }
	        return m_result;
	    }
	
	
	    vector<int> inorderTraversal(TreeNode *root) {
	        if (root) {
	            inorderTraversal(root->left);
	            m_result.push_back(root->val);
	            inorderTraversal(root->right);
	        }
	        return m_result;
	    }
	
	    vector<int> postorderTraversal(TreeNode *root) {
	        if (root) {
	            postorderTraversal(root->left);
	            postorderTraversal(root->right);
	            m_result.push_back(root->val);
	        }
	        return m_result;
	    	}
	
		private:
	    vector<int> m_result;
	};


## 二叉树的非递归遍历 ##
	class Solution {
	public:
		// 每一个节点都会被访问两次，第一次访问该节点时，先访问该节点的所有左子节点,
		// 再访问该节点的所有右子节点，将左右节点都访问完之后，该节点又出现在栈顶，此时为第二次访问，则可访问该节点，访问执行完毕
		// 体现在代码上即，先把栈顶元素取出，如果该节点是第一次访问，则将右节点入栈（如果有），然后将左节点入栈（如果有）。否则该节点是第二次访问，其左右节点均已访问过，则可以访问该节点。则访问顺序最终为左--根--右
	    vector<int> postorderTraversal(TreeNode *root) {
	        vector<int> result;
	        if (root == nullptr)
	            return result;
	        stack<pair<TreeNode *, bool>> util_stack;
	        util_stack.push(make_pair(root, false));
	        while (!util_stack.empty()) {
	            auto top_pair = util_stack.top();
	            util_stack.pop();
	
	            if (top_pair.second) {
	                result.push_back(top_pair.first->val);
	            } else {
	                util_stack.push(make_pair(top_pair.first, true));
	                if (top_pair.first->right) {
	                    util_stack.push(make_pair(top_pair.first->right, false));
	                }
	                if (top_pair.first->left) {
	                    util_stack.push(make_pair(top_pair.first->left, false));
	                }
	            }
	        }
	        return result;
	    }
	
	    void push_all_left(stack<TreeNode *> &util_stack, TreeNode *node) {
	        while (node) {
	            util_stack.push(node);
	            node = node->left;
	        }
	    }
	
		// 先将所有左节点入栈，直到当前节点没有左节点，此时栈中元素的节点的左节点都已入栈，该节点为栈顶的元素，
		// 访问该节点后，对其右节点重复该步骤，则能保证访问顺序为左--根--右
	    vector<int> inorderTraversal(TreeNode *root) {
	        vector<int> result;
	        stack<TreeNode *> util_stack;
	        push_all_left(util_stack, root);
	        while (!util_stack.empty()) {
	            auto top_node = util_stack.top();
	            util_stack.pop();
	            result.push_back(top_node->val);
	            push_all_left(util_stack, top_node->right);
	        }
	        return result
	    }
	
	    vector<int> preorderTraversal(TreeNode *root) {
	        vector<int> result;
	        if (root == nullptr)
	            return result;
	        stack<TreeNode *> util_stack;
	        util_stack.push(root);
	        while (!util_stack.empty()) {
	            auto top_node = util_stack.top();
	            util_stack.pop();
	            result.push_back(top_node->val);
	            if (top_node->right) {
	                util_stack.push(top_node->right);
	            }
	            if (top_node->left) {
	                util_stack.push(top_node->left);
	            }
	        }
	        return result;
	    }

		vector<vector<int>> levelOrder(TreeNode* root) {
	        vector<vector<int>> result;
	        if (root == nullptr)
	            return result;
	        queue<pair<TreeNode*, size_t >> util_queue;
	        util_queue.push(make_pair(root, 0));
	        while (!util_queue.empty()) {
	            auto node = util_queue.front();
	            util_queue.pop();
	
	            size_t level = node.second;
	            if (level == result.size())
	                result.push_back(vector<int>());
	
	            result[level].push_back(node.first->val);
	            if (node.first->left)
	                util_queue.push(make_pair(node.first->left, level + 1));
	            if (node.first->right)
	                util_queue.push(make_pair(node.first->right, level + 1));
	        }
	        return result;
	    }
	};