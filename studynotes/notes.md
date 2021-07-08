## 关键词

**HMAC** 基于hash的消息认证码  ,在做hash操作的同时混入了HMAC密钥,因此只有拥有HMAC密钥才能得到正确的HMAC值.在TPM中主要用作证明用户是否拥有对TPM资源实体的使用权限.

举例:一个TPM中的资源对象(签名密钥)的使用权限可能与一个HMAC的签名密钥有关,这个密钥是用户和TPM都知道的.用户在是用这个TPM对象之前需要首先构造一个命令消息,然后由对象的权限信息派生出HMAC密钥,接着使用HMAC密钥对命令消息做HMAC操作。当TPM收到命令消息和前述的HMAC操作结果之后，使用同样的方法产生HMAC结果。如果两个HMAC值相同就说明命令没有在传输过程中被修改，即命令消息具备完整性。并且命令的发送者确实知道与这个TPM对象关联的HMAC密钥，这意味者命令发送者是经过授权的。这就是一个基于HMAC的授权过程。

**KDF** 密钥生成函数,TPM被设计成可以通过一个秘密信息生成不同的密钥。这个秘密信息称作种子，而算法通过密钥生成函数和这个种子来生成相对应的密钥。TPM就是通过这样的方式来生成对称和非对称密钥的。具体的实现上来说，就是用种子（seed）作为HMAC密钥对一些数据（具体算法相关）做HMAC操作，其结果就是密钥。

**认证或授权凭证** 数据在进出TPM的过程中，HMAC保证了数据的完整性。虽然完整性保证了，但如果数据是以明文的方式存储在TPM之外的地方还是会有问题。这个时候常常需要对称加密密钥对原始数据进行加密以后再送到TPM之外。

**对称加密密钥**  对称加密密钥是一种应用于对称加密算法的密钥。对称加密算法使用相同的密钥做加解密操作。HMAC操作也是一样，HMAC操作对象是数据的哈希，对称加密算法是数据本身.

在TPM中有三种不同的应用:

- 保证TPM数据的机密性,一般来说对称密钥不会存储在TPM以外的地方,所以只有TPM可以访问和创建他们.但是当因为存储控制受限的,而将对称密钥存储到TPM之外时,他首先会被只有TPM能访问的对称密钥加密.
- 加密进出TPM的通讯数据.此时的对称密钥由消息发送者和TPM都知道的秘密信息派生而来.通讯过程中的数据都将会被这个密钥加密.
- 把TPM当作一个对称加解密协处理器。因为TPM完全具备对称加解密的功能，这是完全可行的。你可以将一个对称密钥导入TPM，然后让TPM使用这个密钥来对数据加密。因为TPM的运算速度很慢，所以单纯的加解密功能通常用于少量数据加密。

**对称密钥的模式**  对称加密算法通常用于对成块的大量数据加密.但当数据已成块的模式被加密的时候,需要解决下面两个问题:

- 如果数据块仅仅使用密钥来加密，这叫做ECB模式。因为使用这种模式时，相同的数据快产生的结果总是一样的。当一个位图以ECB的模式被加密时最终的结果是颜色变化（wiki）。这对于大量的数据没有作用，因为数据量变大以后总会出现相同的数据内容，所以最终的密文会有一定程度的体现，这是不安全的.(TPM还支持其他的模式：CBC，CFB，OFB，CTR)
- 有一些模式要求输出必须是算法要求的成块大小，CBC就是这样。如果输入的数据大小与块大小不是整数比例关系，算法将会填充原有的数据块来满足要求。所以这就造成了输出数据大小比输入数据大的情况。对于有些应用来说这不是问题，但是不适合一些明确要求输入输出大小必须一致的应用，比如TPM命令数据流的加密。

这就引出一个非常重要但常常被忽略的问题。加密仅仅提供保密性保护，但是不能提供完整性和认证功能。为了实现后者，TPM会对加密后的数据做HMAC操作而不是依赖加密后的数据看起来是不是对的。因此在实际应用中，对数据的解密总是在验证HMAC之前。

另外，机密并不能证明消息是最近产生的，举例来说有可能是很久之前的一次相同的操作中产生的（回忆以下前面介绍的防止重放）。这时候就需要nonce了

**Nonce** 只能使用一次的随机数,用来阻止重放攻击,为了保证消息不被有效地重复利用,消息的接受者会产生一个nonce发送给消息发送者,发送者将nonce包含在消息中,因为消息发送者不能预测接收者如何选择nonce,所以发送无法事先准备好一个有效的消息.nonce并不是简单地包含在消息中,当用户向TPM发送命令并接收到响应后，Nonce同时也结合HMAC向用户证明用户接收的结果确实来自于真实的TPM。

简述在TPM中的流程:  在TPM中,nonce会被包含在HMAC的计算中.消息涉及的操作完成之后nonce值将会被改变,攻击者如果想继续利用原来的nonce的话,TPM再次验证HMAC将会失败,因为TPM完成了这个消息的操作之后就会对这个nonce进行'遗忘',所以再次验证失败才是正常现象.

**非对称密钥** 非对称密钥主要用于数字身份和密钥管理中,非对称密钥分为两部分,只有一方可以知道的私钥和可以对所有人公布的公钥. 

常见用法: 在加解密应用中，公钥加密数据，私钥解密。在数字签名应用时则相反，私钥用于签名，公钥用于验签。我们将分别介绍其中的细节。

常见非对称密钥算法:

- RSA : 将大数转换为大素数的函数作为单项函数.
  - RAS用于密钥加密: 得到对方的公钥(很容易),加密即可,私钥拥有者解密. 但实际应用中很少用到非对称密钥做加密操作,一般都是用对称密钥加密数据,然后对对称密钥进行加密.这样接受者首先用私钥解密对称密钥,对称密钥解密数据信息. 不用非对称密钥加密的原因主要是非对称加密算法很慢.
  - RSA用于数字签名: 数字签名和HMAC很像,但是它有额外的特性.签名者用私钥对数据进行加密操作就得到了一个数字签名,验签就是用公钥解密签名数据. 数字签名可以保证消息的完整性和真实性,HMAC也有相同的功能,但是基于非对称算的数字签名有着更多的特性,首先,就像上边讲的,HMAC的消息只有拥有HMAC密钥的人才能验证,而非对称算法的因为公钥是公开的,所以是一对多的关系. 并且 签名理论上只有拥有私钥的人才能生成,所以消息接受者使用某一个公钥验证一个签名的时候,第三方就会认为只有拥有私钥的拥有者可以产生签名(不可抵赖性).但是HMAC就做不到这一点,因为HMAC的密钥是双方同时知道的一个秘密,接受者完全可以靠自己生成一个,然后说是对方签名的.  常见的使用模式为: 签名者先对消息hash操作,然后得到消息的摘要,在对摘要签名,接受者接收到消息时,首先会公钥解密出消息的摘要,然后使用和签名者相同的hash算法对消息再做一次摘要,对比摘要值. 这种签名之所以有效,是由于安全hash算法特有的属性,攻击者不能构造出一个和元消息相同摘要的不同消息.
- 


