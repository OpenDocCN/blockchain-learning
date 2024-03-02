# 第三十四章 【IPFS 一问一答】解析 IPFS 应用层 IPLD 之 merkle-dag

# 34\. 【IPFS 一问一答】解析 IPFS 应用层 IPLD 之 merkle-dag

具有 merkle-link 的对象形成一个有方向的图（Merkle-graph），并且可以算作非闭环的(如果加密 Hash 函数的属性保持不变，即 merkle-dag)。 因此，所有使用 merkle-linking（merkle-graph）的图必然也是有向无环图（DAG），因此 merkle-dag。