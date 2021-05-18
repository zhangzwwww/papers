EnclaveCache: A Secure and Scalable Key-value Cache in Multi-tenant Clouds using Intel SGX

### Introduction

#### Problem

Security of user data stored in multi-tenant clouds.

Two risks:

* multi-tenant environment may expose user's sensitive data to other co-located tenants.
* cloud provider can not considered trusted.

#### EnclaveCache Design

* Allow multiple tenant to share a single cache instance and each tenant gets a separate enclave as a secret container.
* Each enclave is responsible for initiating a TLS connection with a specfic tenant, and requesting a unique encryption key from that tenant through the SGX  protocol. 
The key is guarded by the enclave from any unauthorized access including other tenants and privileged admins.
* Plain-text data only stays inside enclaves to process and data is encrypted once it leaves the enclave.
* Redesign the key-value cache into trusted and untrusted parts. Only sensitive information and the critical code is stored inside the enclave.

### Architecture

#### Overall

Each tenant connects to a dedicated secret encalve through the client library.

To ensure the confidentiality of tenant's data during hte network transmission, the secret enclave maintains a TLS connection with the client library. The TLS connection is terminated inside the enclave to avoid unauthorized accesses to the plain-text data after the termination of the encrypted channel.

在encalve 中的 Encryption Engine 负责给 TLS 服务器传来的请求进行加密，然后将加密的 key-value 值转发到系统中不受信任的部分。

Encryption Engine 使用的密钥是通过 KRM 从 KRM 中获得的。KDC 运行在一个 SGX 中（可以运行在客户端或者一个信任的服务器中）KDC 包含 APM 和 KSM

KSM 保存着不同租户的密钥。 APM 是 KDC 中的主要组件，当接到了 KRM 的一个请求，APM 会要求 KRM 证明他们的可信性同时通过ECDH密钥交换协议建立一个安全通道来提供密钥。

#### 密钥分发和管理

1. KRM 要求 APM 去提供密钥。APM 要求 KRM 证实自己（例如应用是跑在 enclave 中的）
2. KRM 生成一个报文来回应 APM 的 challenge， 里面包含了描述 enclave 的内容的哈希和一个之后用来通信的公钥。
3. 报文转发给 QE 来验证。验证之后 QE 使用自己的 EPID key 来给报文进行签名同时生成了一个 QUOTE。将这个 QUOTE 返回给KRM
4. KRM 将 QUOTE 传给 APM，APM 调用 API 来验证 QUOTE 的签名。如果验证成功，APM 解析QUOTE中的哈希值来确认代码是运行在enclave中的
5. 验证成功后，获得对称密钥，之后 APM 和 KRM 能够进行通信
6. APM 拿到在 KSM 中存储的密钥并将密钥加密后传给 KRM， KRM 能够使用公钥解密

#### 查询阶段

在 TLS 终止后，请求会被解密为文本然后交给 Encryption Engine 去处理

在一个查询处理的过程中有两个不同的地方发生加密：

* 整个消息在TLS 客户端和 TLS 服务端之间传输的时候使用 TLS 自动生成的 session key 进行加密。
* Encryption Engine 会讲消息中敏感的去余使用 KRM 请求到的密钥进行加密。

Encryption Engine 先将信息进行反序列化，然后提取其中的敏感信息。然后将信息进行加密，通过SHA256得到的 Initialization Vector 以及 Message Authentication Code 会拼接在密文的后面。

由于EnclaveCache数据库位于不受信任的环境中，因此有必要将每个值与其对应的密钥绑定在一起。对于每个键值对，EnclaveCache会将键的哈希值分配给其对应的值，然后对新生成的值执行加密。 通过这种方法，安全区可以验证从不受信任的请求处理程序返回的值是否属于所请求的密钥。 对所有敏感字段进行加密后，将根据消息协议重新组合消息，以使每个敏感字段都被其相应的密文替换。 最后，消息被转发到请求处理程序进行处理。

验证从请求处理程序返回的响应，以确保指针参数指向具有预期长度和数据类型的有效响应缓冲区。 然后，以与请求处理相似的过程处理响应，但顺序相反：加密引擎首先执行反序列化以提取先前已加密的敏感字段，然后解密每个字段，并将响应安全地返回给客户端使用TLS。
