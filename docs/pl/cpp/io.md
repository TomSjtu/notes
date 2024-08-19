# I/O流

C的`printf()`和`scanf()`不能处理错误，也不是类型安全的。C++通过流的机制提供了更精良的输入输出方法(但是性能开销也比C方式要高)，流是一种灵活的、面向对象的I/O方法。

## ACM输入输出

在笔试的时候，ACM输入输出(即需要自己定义输入与输出)是常见的模式。以下列举常见的用法：

1. 只有一个或几个输入

    输入样例:

    ```
    3 5 7
    ```

    模板：

    ```C
    int a, b, c;
    cin >> a >> b >> c;
    ```

2. 先给行数T，再给出T行

    输入样例:

    ```
    3
    3 5 7
    6 8 9
    12 9 5
    ```

    模板：

    ```C
    int T;
    cin >> T;
    while(T--) {
        int a, b, c;
        cin >> a >> b >> c;
        //处理输入
    }
    ```

3. 整行读取字符串

    输入样例：

    ```
    Hello world
    ```

    模板：

    ```C
    string s;
    getline(cin, s);
    //处理输入
    ```

4. 读取以逗号分开的字符串

    输入样例：

    ```
    one,two,three
    ```

    模板：

    ```C
    string line;
    getline(cin, line);
    stringstream ss(line);
    string token;
    while(getline(ss, token, ',')){
        //处理token
    }
    ```

5. 构造链表

    ```C
    #include <iostream>
    using namespace std;
    
    struct ListNode{
        int val;
        ListNode* next;
        ListNode(int x) : val(x), next(nullptr) {}
        ListNode(int x, ListNode* n) : val(x), next(n) {}
    }

    int main(){
        ListNode *dummy = new ListNode(-1);
        ListNode *pre = dummy;
        ListNod *cur = nullptr;
        int num;
        while(cin>>num){
            cur = new ListNode(num);
            pre->next = cur;
            pre = cur;
        }

        cur = dummy->next;

        while(cur){
            cout<<cur->val<<" ";
            cur = cur->next;
        }

        return 0;
    }
    ```

6. 构造二叉树


    ```C
    #include <iostream>
    #include <vector>
    #include <queue>

    using namespace std;

    //定义树节点
    struct TreeNode
    {
        int val;
        TreeNode* left;
        TreeNode* right;
        TreeNode():val(0),left(nullptr),right(nullptr){}
        TreeNode(int _val):val(_val),left(nullptr),right(nullptr){}
        TreeNode(int _val,TreeNode* _left,TreeNode* _right):val(0),left(_left),right(_right){}
    };

    //根据数组生成树
    TreeNode* buildTree(const vector<int>& v)
    {
        vector<TreeNode*> vTree(v.size(),nullptr);
        TreeNode* root = nullptr;
        for(int i = 0; i < v.size(); i++)
        {
            TreeNode* node = nullptr;
            if(v[i] != -1)
            {
                node = new TreeNode(v[i]);
            }
            vTree[i] = node;
        }
        root = vTree[0];
        for(int i = 0; 2 * i + 2 < v.size(); i++)
        {
            if(vTree[i] != nullptr)
            {
                vTree[i]->left = vTree[2 * i + 1];
                vTree[i]->right = vTree[2 * i + 2];
            }
        }
        return root;
    }

    //根据二叉树根节点层序遍历并打印
    void printBinaryTree(TreeNode* root)
    {
        if(root == nullptr) return;
        vector<vector<int>> ans;
        queue<TreeNode*> q;
        q.push(root);
        while(!q.empty())
        {
            int size = q.size();
            vector<int> path;
            for(int i = 0;i<size;i++)
            {
                TreeNode* node = q.front();
                q.pop();
                if(node == nullptr)
                {
                    path.push_back(-1);
                }
                else
                {
                    path.push_back(node->val);
                    q.push(node->left);
                    q.push(node->right);
                }
            }
            ans.push_back(path);
        }
        
        for(int i = 0;i<ans.size();i++)
        {
            for(int j = 0;j<ans[i].size();j++)
            {
                cout << ans[i][j] << " ";
            }
            cout << endl;
        }
        return;
    }

    int main()
    {
        // 验证
        vector<int> v = {4,1,6,0,2,5,7,-1,-1,-1,3,-1,-1,-1,8};
        TreeNode* root = buildTree(v);
        printBinaryTree(root);
        
        return 0;
    }


    ```

## 流的概念

流可以看成是数据的缓冲，`cout`和`cin`是C++在std名称空间预定义的流实例。缓冲流不会讲数据立即发送，而是先存储在缓冲区，然后以块的形式发送。非缓冲流则会立即将数据发送至目的地。缓冲的目的通常是为了提高性能。`flush()`方法可以强制刷新缓冲流的缓冲区。

| 流 | 说明 |
| --- | --- |
| cin | 标准输入流 |
| cout | 缓冲的标准输出流 |
| cerr | 非缓冲的错误输出流 |

这些流还支持宽字符版本，以w开头：wcin、wcout、wcerr。宽字符用于字符数多于英语的语言，比如中文。

!!! note

    GUI程序通常没有控制台，因此不要假定这些流存在，即便写入了数据，用户也无法看到。

## 文件流

`std::ofstream`和`std::ifstream`分别用于打开文件进行写入和读取。这两个类在<fstream\>头文件中定义。

文件流的参数有：

| 参数 | 说明 |
| --- | --- |
| ios_base::app | 在末尾追加 |
| ios_base::binary | 以二进制模式打开 |
| ios_base::in | 从开头开始读取 |
| ios_base::out | 从开头开始写入 |
| ios_base::trunc | 删除已有数据 |

`ifstream`自动包含`ios_base::in`，`ofstream`自动包含`ios_base::out`。且二者会自动关闭底层文件，因此不需要显示调用`close()`。

### 文件流的定位

文件流支持定位操作，`seekp()`和`seekg()`分别用于定位写入和读取的文件流。位置的类型为`std::streampos`，偏移量的类型为`std::streamoff`，两种类型都以字节计数。

预定义的3个位置如下图：

| 位置 | 说明 |
| --- | --- |
| ios_base::beg | 流开头 |
| ios_base::end | 流末尾 |
| ios_base::cur | 流当前位置 |

!!! example "定位流"

    ```CPP
    //定位到输出流的开头
    std::streampos begPos = outStream.seekp(ios_base::beg);

    //定位到输入流的末尾
    std::streampos endPos = inStream.seekg(ios_base::end);
    ```

`tellp()`和`tellg()`分别用于获取写入和读取的文件流的当前位置。



## filesystem库

C++17引入了`std::filesystem`库，用于处理文件系统。

### 路径

路径可以是绝对的，也可以是相对的：`filesystem::path p {"test.txt"}`。

可以使用`append()`函数或者`operator+=`将字符串拼接到路径中，与平台相关的路径分隔符会自动插入。而`concat()`函数或`operator+=`则不会插入分隔符。

基于范围的for循环可以遍历路径的不同组件，比如：

```CPP
filesystem::path p {R"(C:\Foo\Bar)"};
for(const auto &component: p){
    cout<<component<<endl;>>
}
```

Windows上的输出如下：

```
"C:"
"\\"
"Foo"
"Bar"
```

### 目录

`create_directory()`用于创建一个新目录。

