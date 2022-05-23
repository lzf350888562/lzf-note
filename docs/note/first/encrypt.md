# 编码和加密

# 一、URLENCODE

## （1）为什么使用

1、是因为当字符串数据以url的形式传递给web服务器时,字符串中是不允许出现空格和特殊字符的。

2、因为 url 对字符有限制，比如把一个邮箱放入 url，就需要使用 `urlencode` 函数，因为 url 中不能包含 `@` 字符。

3、url转义其实也只是为了符合url的规范而已。因为在标准的url规范中中文和很多的字符是不允许出现在url中的。

## （2）转换规则

> urlencode：

返回字符串，此字符串中除了-_.之外的所有非字母数字字符都将被替换成百分号（%）后跟两位十六进制数，空格则编码为加号（+）
按照每个字符对应的字符编码，不是符合我们范围的，统统的转化为%的形式也就是了。自然也是16进制的形式。

> 那哪些字符是需要转化的呢

ASCII 的控制字符
这些字符都是不可打印的，自然需要进行转化。

> 什么是可打印字符？

在ASCII码中规定，0-31、128这33个字符属于控制字符，32-127这95个字符属于可打印字符，也就是说网络传输只能传输这95个字符，不在这个范围内的字符无法传输。那么该怎么才能传输其他字符呢？其中一种方式就是使用Base64。

> 一些非ASCII字符

这些字符自然是非法的字符范围。转化也是理所当然的了。

> 一些保留字符

很明显最常见的就是“&”了，这个如果出现在url中了，那你认为是url中的一个字符呢，还是特殊的参数分割用的呢？

> 就是一些不安全的字符了。

例如：空格。为了防止引起歧义，需要被转化为“+”。

> 和编码无关

通过urlencode的转化规则和目的，我们也很容易的看出，urleocode是基于字符编码的。同样的一个汉字，不同的编码类型，肯定对应不同的urleocode的串。gbk编码的有gbk的encode结果。

apache等服务器，接受到字符串后，可以进行decode，但是还是无法解决编码的问题。编码问题，还是需要靠约定或者字符编码的判断解决。

因此，urleocode只是为了url中一些非ascii字符，可以正确无误的被传输，至于使用哪种编码，就不是eocode所关心和解决的问题了。

## （3）java 实现

```java
public class UrlEncode {
    public static void main(String[] args) throws UnsupportedEncodingException {
        String encode = URLEncoder.encode("http://itnanls.cn/中文  ，。？", "UTF-8");
        System.out.println(encode);

        String decode = URLDecoder.decode(encode, "UTF-8");
        System.out.println(decode);
    }
}
```

# 二、Base64编解码

## 1、Base64是什么

- Base64是一种用64个字符来表示任意二进制数据的方法。
- 它是一种编码方式，而非加密方式。
- 它通过将二进制数据转变为64个“可打印字符”，完成了数据在HTTP协议上的传输。

## 2、使用场景

Base64一般用于在HTTP协议下传输二进制数据，由于HTTP协议是文本协议，所以在HTTP协议下传输二进制数据需要将二进制数据转换为字符数据。然而直接转换是不行的。因为网络传输只能传输可打印字符。

> 什么是可打印字符？

在ASCII码中规定，0-31、128这33个字符属于控制字符，32-127这95个字符属于可打印字符，也就是说网络传输只能传输这95个字符，不在这个范围内的字符无法传输。那么该怎么才能传输其他字符呢？其中一种方式就是使用Base64。

既然可打印的只有95，而比95小的整数最大的2的次方的数字就是64了。

## 3、加密规则

关于这个编码的规则：

- 首先将待编码的内容转换成8位二进制，每3个字符为一组；

- 如果编码前的长度是3n+1，编码后的内容最后面补上2个 ‘=’，如果编码前的长度是3n+2，编码后 的内容最后面补上1个 ‘=’。

- 再将每一组的二进制内容拆分成6位的二进制，不足6位的后面补足0；

- 每个6进制的数字前面补足0，保证变成8位二进制；

- 将补足后的内容根据base64编码表转换成base64内容输出；

（不足四个字符的时候会用 ‘=’ 来补足，下面会说明）

> 例子

编码前 “hb”

**1.**根据ascii码转换成8位二进制，3个为一组：

```
  01101000,01100010
```

**2.**编码前长度是3n+2，所以后面补1个 ‘=’：

```
  01101000,01100010,=
```

**3.**拆分成6位二进制，不足6位的在后面补足0，0010补足变成001000：

```
  011010,000110,001000,=
```

**4.**每个6进制的数字前面补足0：

```
  0011010,00000110,00000010,=
```

**5.**根据base64编码表输出：

```
  aGI=
```

Table 1: The Base64 Alphabet

| **索引** | **对应字符** | **索引** | **对应字符** | **索引** | **对应字符** | **索引** | **对应字符** |
| ------ | -------- | ------ | -------- | ------ | -------- | ------ | -------- |
| 0      | **A**    | 17     | **R**    | 34     | **i**    | 51     | **z**    |
| 1      | **B**    | 18     | **S**    | 35     | **j**    | 52     | **0**    |
| 2      | **C**    | 19     | **T**    | 36     | **k**    | 53     | **1**    |
| 3      | **D**    | 20     | **U**    | 37     | **l**    | 54     | **2**    |
| 4      | **E**    | 21     | **V**    | 38     | **m**    | 55     | **3**    |
| 5      | **F**    | 22     | **W**    | 39     | **n**    | 56     | **4**    |
| 6      | **G**    | 23     | **X**    | 40     | **o**    | 57     | **5**    |
| 7      | **H**    | 24     | **Y**    | 41     | **p**    | 58     | **6**    |
| 8      | **I**    | 25     | **Z**    | 42     | **q**    | 59     | **7**    |
| 9      | **J**    | 26     | **a**    | 43     | **r**    | 60     | **8**    |
| 10     | **K**    | 27     | **b**    | 44     | **s**    | 61     | **9**    |
| 11     | **L**    | 28     | **c**    | 45     | **t**    | 62     | **+**    |
| 12     | **M**    | 29     | **d**    | 46     | **u**    | 63     | **/**    |
| 13     | **N**    | 30     | **e**    | 47     | **v**    |        |          |
| 14     | **O**    | 31     | **f**    | 48     | **w**    |        |          |
| 15     | **P**    | 32     | **g**    | 49     | **x**    |        |          |
| 16     | **Q**    | 33     | **h**    | 50     | **y**    |        |          |

## 4、总结

- 可以将`任意的二进制`数据进行Base64编码。
- 数据加密之后，数据量会`变大`，变大`1/3`左右。
- 编码后有个非常显著的特点，末尾有个=号。
- 可进行`反向解密`。
- Base64编码具有不可读性

## 5、Base64程序

```java
/**
 * @author itnanls
 * @date 2021/3/15
 */
public class Base64Test {

    public static void main(String[] args) {
        // 获取编码器
        Base64.Encoder encoder = Base64.getEncoder();
        byte[] encode = encoder.encode("abc".getBytes());
        System.out.println(new String(encode));

        // 获取解码器
        Base64.Decoder decoder = Base64.getDecoder();
        byte[] decode = decoder.decode(encode);
        System.out.println(new String(decode));
    }
}
```

# 三、hash摘要算法

## 1、MD5

**MD5信息摘要算法**（英语：MD5 Message-Digest Algorithm），一种被广泛使用的[密码散列函数](https://baike.baidu.com/item/密码散列函数/14937715)，可以产生出一个128位（16[字节](https://baike.baidu.com/item/字节/1096318)）的散列值（hash value），用于确保信息传输完整一致。MD5由美国密码学家[罗纳德·李维斯特](https://baike.baidu.com/item/罗纳德·李维斯特/700199)（Ronald Linn Rivest）设计，于1992年公开，用以取代[MD4](https://baike.baidu.com/item/MD4/8090275)算法。这套算法的程序在 RFC 1321 标准中被加以规范。1996年后该算法被证实存在弱点，可以被加以破解，对于需要高度安全性的数据，专家一般建议改用其他算法，如[SHA-2](https://baike.baidu.com/item/SHA-2/22718180)。2004年，证实MD5算法无法防止碰撞（collision），因此不适用于安全性认证，如[SSL](https://baike.baidu.com/item/SSL/320778)公开密钥认证或是[数字签名](https://baike.baidu.com/item/数字签名/212550)等用途。

```java
public static void main(String[] args) throws Exception {
    MessageDigest md5 = MessageDigest.getInstance("MD5");
    byte[] digest = md5.digest("123".getBytes());
    System.out.println(digest.length);
}
16
```

## 2、SHA1

**SHA-1**（英语：Secure Hash Algorithm 1，中文名：安全散列算法1）是一种[密码散列函数](https://baike.baidu.com/item/密码散列函数)，[美国国家安全局](https://baike.baidu.com/item/美国国家安全局)设计，并由美国国家标准技术研究所（NIST）发布为联邦数据处理标准（FIPS）。SHA-1可以生成一个被称为消息摘要的160[位](https://baike.baidu.com/item/位)（20[字节](https://baike.baidu.com/item/字节)）散列值，散列值通常的呈现形式为40个[十六进制](https://baike.baidu.com/item/十六进制/4162457)数。

> SHA 家族

正式名称为 SHA 的家族第一个成员发布于 1993年。然而人们给它取了一个非正式的名称 SHA-0 以避免与它的后继者混淆。两年之后， SHA-1，第一个 SHA 的后继者发布了。 另外还有四种变体，曾经发布以提升输出的范围和变更一些细微设计: SHA-224, SHA-256, SHA-384 和 SHA-512 (这些有时候也被称做 SHA-2)：

```java
public static void main(String[] args) Exception {
        MessageDigest sha1 = MessageDigest.getInstance("SHA1");
        byte[] digest = sha1.digest("123".getBytes());
        System.out.println(digest.length);
}

20
```

## 3、SHA256

SHA256算法使用的哈希值长度是256位。

```java
public static void main(String[] args) throws Exception {
    MessageDigest sha256 = MessageDigest.getInstance("SHA-256");
    byte[] digest = sha256.digest("123".getBytes());
    System.out.println(digest.length);
}

32
```

## 4、SHA512

算法使用的哈希值长度是512位。

```java
public static void main(String[] args) throws Exception {
    MessageDigest sha512 = MessageDigest.getInstance("SHA-512");
    byte[] digest = sha512.digest("123".getBytes());
    System.out.println(digest.length);
}

64
```

# 四、对称加密

## 1、DES算法

DES 加密算法是一种 分组密码，以 64 位为 分组对数据 加密，它的 密钥长度 是 56 位，加密解密 用 同一算法。

数据加密标准（Data Encryption Standard），简称DES，是由IBM公司提交，美国政府于1977年1月5日颁布的一种加密算法。

- DES的设计目标是，用于加密保护静态存储和传输信道中的数据，安全使用10~15年。

- DES综合运用了置换、代替、代数等多种密码技术。它设计精巧、实现容易、使用方便，堪称是适应计算机环境的近代分组密码的一个典范。DES的设计充分体现了Shannon所阐述的设计密码的思想，标志着密码的设计与分析达到了新的水平。

- DES是一种分组密码。明文、密文和密钥的分组长度都是64位。

- DES是面向二进制的密码算法。因而能够加解密任何形式的计算机数据。

- DES是对合运算，因而加密和解密共用同一算法，从而使工程实现的工作量减半。

- DES的密码结构属于Feistel结构。

### （1）DES的加密过程

- 64位密钥经子密钥产生算法产生出16个48位子密钥：K1,K2,...,K16，分别供第1次，第2次，...，第16次加密迭代使用。

- 64位明文首先经过初始置换IP，将数据打乱重新排列并分成左右两半，左边32位构成L0，右边32位构成R0。

- 第i次加密迭代：由轮函数f实现子密钥Ki对Ri-1的加密，结果为32位的数据组f ( Ri-1 , Ki )。f ( Ri-1 , Ki )再与Li-1模2相加，又得到一个32位的数据组Li-1 ⊕ f ( Ri-1 , Ki )。以Li ⊕ f ( Ri-1 , Ki )作为下一次加密迭代的Ri，以Ri-1作为下一次加密迭代的Li ( i = 1,2,...,16)。

- 按照上一步的规则进行16次加密迭代。

- 第16次加密迭代结束后，以R16为左，L16为右，合并产生一个64位的数据组。再经过逆初始置换IP-1，将数据重新排列，便得到64位密文。

------

### （2）DES的解密过程

- 64位密钥经子密钥产生算法产生出16个48位子密钥：K1,K2,...,K16，分别供第1次，第2次，...，第16次解密迭代使用。

- 64位密文首先经过初始置换IP，将数据打乱重新排列并分成左右两半，左边32位构成R16，右边32位构成L16。

- 第17-i次解密迭代：由轮函数f实现子密钥Ki对Li的解密，结果为32位的数据组f ( Li , Ki )。f ( Li , Ki )再与Ri模2相加，又得到一个32位的数据组Ri ⊕ f ( Li , Ki )。以Ri ⊕ f ( Li , Ki )作为下一次解密迭代的Li-1，以Li作为下一次解密迭代的Li-1 ( i = 16,15,...,1)。

- 按照上一步的规则进行16次解密迭代。

- 第16次解密迭代结束后，以L0为左，R0为右，合并产生一个64位的数据组。再经过逆初始置换IP-1，将数据重新排列，便得到64位明文。

### （3）java实现

```java
/**
 * @author itnanls
 * @date 2021/3/15
 */
public class DES {

    /**
     * 算法秘钥
     */
    private static final byte[] DES_KEY = {11, 21, 1, -110, 82, -32, -85, -128, -65 ,44,-2};

    /**
     * 算法名字
     */
    private static final String ALGORITHM_NAME = "DES";
    /**
     * 数据加密，算法（DES）
     * @param data
     * 要进行加密的数据
     * @return 加密后的数据
     */
    public static byte[] encryptDes(byte[] data) {
        try {
            // DES算法要求有一个可信任的随机数源
            DESKeySpec desKey = new DESKeySpec(DES_KEY);
            // 创建一个密匙工厂，然后用它把DESKeySpec转换成一个SecretKey对象
            SecretKeyFactory keyFactory = SecretKeyFactory.getInstance(ALGORITHM_NAME);
            SecretKey key = keyFactory.generateSecret(desKey);
            // 加密对象
            Cipher cipher = Cipher.getInstance(ALGORITHM_NAME);
            cipher.init(Cipher.ENCRYPT_MODE, key);
            // 加密
            return cipher.doFinal(data);
        } catch (Exception e) {
            throw new RuntimeException("加密错误，错误信息：", e);
        }
    }

    /**
     * 解密
     * @param cryptData
     * @return
     */
    public static byte[] decryptBasedDes(byte[] cryptData) {
        try {
            // DES算法要求有一个可信任的随机数源
            DESKeySpec desKey = new DESKeySpec(DES_KEY);
            // 创建一个密匙工厂，然后用它把DESKeySpec转换成一个SecretKey对象
            SecretKeyFactory keyFactory = SecretKeyFactory.getInstance(ALGORITHM_NAME);
            SecretKey key = keyFactory.generateSecret(desKey);
            // 解密对象
            Cipher cipher = Cipher.getInstance(ALGORITHM_NAME);
            cipher.init(Cipher.DECRYPT_MODE, key);
            // 把字符串解码为字节数组，并解密
            return cipher.doFinal(cryptData);
        } catch (Exception e) {
            throw new RuntimeException("解密错误，错误信息：", e);
        }
    }

    public static void main(String[] args) {

        String data = "itnanls is so shuai";

        byte[] result = encryptDes(data.getBytes());
        System.out.println("加密后：" + Base64.getEncoder().encode(result));

        // DES数据解密
        byte[] source = decryptBasedDes(result);
        System.err.println("解密后:" + new String(source));
    }
}
```

## 2、AES

### （1）介绍

密码学中的高级加密标准（Advanced Encryption Standard，AES），又称Rijndael[加密法](https://baike.baidu.com/item/加密法/9684302)，是[美国联邦政府](https://baike.baidu.com/item/美国联邦政府/8370227)采用的一种区块加密标准。

这个标准用来替代原先的[DES](https://baike.baidu.com/item/DES)（Data Encryption Standard），已经被多方分析且广为全世界所使用。经过五年的甄选流程，高级加密标准由美国国家标准与技术研究院 （NIST）于2001年11月26日发布于FIPS PUB 197，并在2002年5月26日成为有效的标准。2006年，高级加密标准已然成为对称密钥加密中最流行的算法之一 [1] 。

### （2）java实现

```java
/**
 * /**
 *
 * @author itnanls
 * @date 2021/3/15
 */
public class AESUtil {

    /**
     * 算法秘钥
     */
    private static final byte[] AES_KEY = {11, 21, 1, -110, 82, -32, -85,
            -128, -65, 44, -2, 23, 12, 32, 23, 45};

    /**
     * 算法名字
     */
    private static final String ALGORITHM_NAME = "AES";

    public static byte[] encrypt(byte[] src) {
        try {
            // 根据秘钥和加密算法，生成秘钥对象
            SecretKeySpec secretKeySpec = new SecretKeySpec(AES_KEY, ALGORITHM_NAME);
            // 得到加密器的实例
            Cipher cipher = Cipher.getInstance(ALGORITHM_NAME);
            // 初始化密码器
            cipher.init(Cipher.ENCRYPT_MODE, secretKeySpec);
            return cipher.doFinal(src);
        } catch (Exception e) {
            e.printStackTrace();
        }
        return null;
    }

    public static byte[] decrypt(byte[] src) {
        try {
            SecretKeySpec secretKeySpec = new SecretKeySpec(AES_KEY, ALGORITHM_NAME);
            Cipher cipher = Cipher.getInstance(ALGORITHM_NAME);
            cipher.init(Cipher.DECRYPT_MODE, secretKeySpec);
            return cipher.doFinal(src);
        } catch (Exception e) {
            e.printStackTrace();
        }
        return null;
    }

    public static void main(String[] args) {

        String src = "itnanls is so cool!";

        byte[] encrypt = encrypt(src.getBytes());
        assert encrypt != null;
        System.out.println(new String(encrypt));

        byte[] decrypt = decrypt(encrypt);
        assert decrypt != null;
        System.out.println(new String(decrypt));
    }
}
```

# 五、非对称加密

## （1）介绍

- RSA加密算法：RSA加密算法是一种非对称加密算法。在公开密钥加密和电子商业中RSA被广泛使用。RSA是1977年由罗纳德·李维斯特（Ron Rivest）、阿迪·萨莫尔（Adi Shamir）和伦纳德·阿德曼（Leonard Adleman）一起提出的。当时他们三人都在麻省理工学院工作。RSA就是他们三人姓氏开头字母拼在一起组成的。

- RSA是目前最有影响力的公钥加密算法，该算法基于一个十分简单的数论事实：将两个大素数相乘十分容易，但那时想要对其乘积进行因式分解却极其困难，因此可以将乘积公开作为加密密钥，即公钥，而两个大素数组合成私钥。公钥是可发布的供任何人使用，私钥则为自己所有，供解密之用。

- 解密者拥有私钥，并且将由私钥计算生成的公钥发布给加密者。加密都使用公钥进行加密，并将密文发送到解密者，解密者用私钥解密将密文解码为明文。

## （2）java实现

```java
/**
 * @author itnanls
 * @date 2021/3/15
 */
public class RSATest {

    private static final String PUBLIC_KEY = "publicKey";

    private static final String PRIVATE_KEY = "privateKey";

    private static final String ALGORITHM_NAME = "RSA";

    private static final Integer KEY_SIZE = 1024;

    public static final RsaKeyPair RSA_KEY_PAIR = new RsaKeyPair();

    /**
     * 存放密钥对的静态内部类
     */
    private static final class RsaKeyPair {
        public RSAPublicKey publicKey;
        public RSAPrivateKey privateKey;
    }

    public static void main(String[] args) throws Exception {

        initKey();

        // 公钥加密，私钥解密，比人加密给我看，就是加密
        byte[] publicKeyEncrypt = encryptByPublicKey("123".getBytes());
        byte[] privateKeyDecrypt = decryptByPrivateKey(publicKeyEncrypt);
        System.out.println(new String(privateKeyDecrypt));
        // 私钥加密，公钥解密，我加密了，别人拿公钥解密看，能证明文件是我的，数字签名
        byte[] privateKeyEncrypt = encryptByPrivateKey("123".getBytes());
        byte[] publicKeyDecrypt = decryptByPublicKey(privateKeyEncrypt);
        System.out.println(new String(publicKeyDecrypt));
    }

    // 初始化密钥对
    public static void initKey(){
        // 实例化密钥对生成器
        KeyPairGenerator keyPairGen = null;
        try {
            keyPairGen = KeyPairGenerator.getInstance(ALGORITHM_NAME);
        } catch (NoSuchAlgorithmException e) {
            e.printStackTrace();
        }

        // 初始化密钥对生成器
        assert keyPairGen != null;
        keyPairGen.initialize(KEY_SIZE);

        // 生成密钥对
        KeyPair keyPair = keyPairGen.generateKeyPair();

        // 公钥
        RSA_KEY_PAIR.publicKey = (RSAPublicKey) keyPair.getPublic();

        // 私钥
        RSA_KEY_PAIR.privateKey = (RSAPrivateKey) keyPair.getPrivate();

    }

    /**
     * 公钥解密
     * @param data
     * 待解密数据
     * @return byte[] 解密数据
     * @throws Exception
     */
    public static byte[] decryptByPublicKey(byte[] data)
            throws Exception {

        // 取得公钥
        // 此类表示根据 ASN.1 类型 SubjectPublicKeyInfo 进行编码的公用密钥的 ASN.1 编码。
        // X.509 标准中定义的 SubjectPublicKeyInfo 语法
        X509EncodedKeySpec x509KeySpec =
                new X509EncodedKeySpec(RSA_KEY_PAIR.publicKey.getEncoded());
        KeyFactory keyFactory = KeyFactory.getInstance(ALGORITHM_NAME);

        // 生成公钥
        PublicKey publicKey = keyFactory.generatePublic(x509KeySpec);

        // 对数据解密
        Cipher cipher = Cipher.getInstance(keyFactory.getAlgorithm());

        cipher.init(Cipher.DECRYPT_MODE, publicKey);

        return cipher.doFinal(data);
    }

    /**
     * 公钥加密
     * @param data
     * 待加密数据
     * @return byte[] 加密数据
     * @throws Exception
     */
    public static byte[] encryptByPublicKey(byte[] data)
            throws Exception {

        // 取得公钥
        X509EncodedKeySpec x509KeySpec =
                new X509EncodedKeySpec(RSA_KEY_PAIR.publicKey.getEncoded());

        KeyFactory keyFactory = KeyFactory.getInstance(ALGORITHM_NAME);

        PublicKey publicKey = keyFactory.generatePublic(x509KeySpec);

        // 对数据加密
        Cipher cipher = Cipher.getInstance(ALGORITHM_NAME);
        cipher.init(Cipher.ENCRYPT_MODE, publicKey);

        // 完成加密
        return cipher.doFinal(data);
    }

    /**
     * 私钥加密
     *
     * @param data
     * 待加密数据
     * @return byte[] 加密数据
     * @throws Exception
     */
    public static byte[] encryptByPrivateKey(byte[] data)
            throws Exception {

        // 取得私钥
        // 该类代表私有密钥的ASN.1编码，根据ASN.1类型PrivateKeyInfo进行编码。
        // PrivateKeyInfo语法在PKCS＃8标准中
        PKCS8EncodedKeySpec pkcs8KeySpec =
                new PKCS8EncodedKeySpec(RSA_KEY_PAIR.privateKey.getEncoded());
        KeyFactory keyFactory = KeyFactory.getInstance(ALGORITHM_NAME);

        // 生成私钥
        PrivateKey privateKey = keyFactory.generatePrivate(pkcs8KeySpec);

        // 初始化加密器
        Cipher cipher = Cipher.getInstance(keyFactory.getAlgorithm());
        cipher.init(Cipher.ENCRYPT_MODE, privateKey);
        // 进行加密
        return cipher.doFinal(data);
    }


    /**
     * 私钥解密
     *
     * @param data
     * 待解密数据
     * @return byte[] 解密数据
     * @throws Exception
     */
    public static byte[] decryptByPrivateKey(byte[] data)
            throws Exception {

        // 取得私钥
        // 该类代表私有密钥的ASN.1编码，根据ASN.1类型PrivateKeyInfo进行编码。
        // PrivateKeyInfo语法在PKCS＃8标准中
        PKCS8EncodedKeySpec pkcs8KeySpec =
                new PKCS8EncodedKeySpec(RSA_KEY_PAIR.privateKey.getEncoded());

        KeyFactory keyFactory = KeyFactory.getInstance(ALGORITHM_NAME);

        // 生成私钥
        PrivateKey privateKey = keyFactory.generatePrivate(pkcs8KeySpec);

        // 对数据解密
        Cipher cipher = Cipher.getInstance(keyFactory.getAlgorithm());

        cipher.init(Cipher.DECRYPT_MODE, privateKey);

        return cipher.doFinal(data);
    }
}
```
