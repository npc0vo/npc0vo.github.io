---
title: 2025hgame
tags:
  - WriteUp
description:
date: 2025-03-05
aliases:
draft:
---
## Week1

### CompressDotNew

gpt秒了 huffman解码

```python
import json

# 哈夫曼树节点类
class HuffmanNode:
    def __init__(self, s=None, a=None, b=None):
        self.s = s  # 叶子节点存储的字符
        self.a = a  # 左子节点
        self.b = b  # 右子节点

# 解析哈夫曼树的JSON结构
def parse_huffman_tree(json_data):
    if "s" in json_data:
        return HuffmanNode(s=json_data["s"])
    else:
        return HuffmanNode(
            a=parse_huffman_tree(json_data["a"]),
            b=parse_huffman_tree(json_data["b"])
        )

# 解码二进制数据
def decode_binary_data(huffman_tree, encoded_data):
    current_node = huffman_tree
    result = []

    for bit in encoded_data:
        if bit == "0":
            current_node = current_node.a
        else:
            current_node = current_node.b

        if current_node.s is not None:
            result.append(current_node.s)
            current_node = huffman_tree

    return bytes(result)

# 解压缩主函数
def decompress(compressed_data):
    # 分割哈夫曼树和编码数据
    huffman_tree_json, encoded_data = compressed_data.split("\n", 1)

    # 解析哈夫曼树
    huffman_tree = parse_huffman_tree(json.loads(huffman_tree_json))

    # 解码二进制数据
    decoded_bytes = decode_binary_data(huffman_tree, encoded_data)

    return decoded_bytes

# 示例使用
if __name__ == "__main__":
    # 读取压缩数据
    with open("enc.txt", "r") as file:
        compressed_data = file.read()

    # 解压缩
    decompressed_data = decompress(compressed_data)

    # 保存解压缩结果
    with open("flag.txt", "wb") as file:
        file.write(decompressed_data)

    print("解压缩完成，结果已保存到 flag.txt")
```

`hgame{Nu-Shell-scr1pts-ar3-1nt3r3st1ng-t0-wr1te-&-use!}`

### Turtle

首先是一个魔改upx壳，直接用esp定律就可以脱壳

然后是两个rc4

第二个魔改了

![[3c843c929bdfb938a4873500bdd6a66a_MD5.png]]

直接动调拿keybox

```
sbox1=[0x16, 0xDF, 0x4C, 0x7B, 0xD5, 0x2C, 0x3F, 0x2D, 0xA5, 0xF6, 0xF0, 0x66, 0x54, 0x5D, 0xB9, 0x26, 0xD3, 0x4F, 0x45, 0xA6, 0x58, 0xCD, 0x12, 0x0B, 0xAC, 0x78, 0x27, 0x36, 0xA3, 0x8D, 0x24, 0xA8, 0x41, 0xFE, 0x2B, 0x03, 0x15, 0xE9, 0x7A, 0x60, 0xF9, 0x3B, 0xDE, 0x9E, 0x1C, 0xC4, 0x3E, 0x86, 0xAD, 0x88, 0x82, 0x33, 0x69, 0x35, 0x17, 0xB7, 0x5A, 0xEB, 0x92, 0x0A, 0xDC, 0x7E, 0x2A, 0x8E, 0x74, 0x59, 0xDB, 0xE2, 0x19, 0xAF, 0x76, 0x4D, 0xA0, 0x5B, 0x71, 0x13, 0xA9, 0x4E, 0xCE, 0x1B, 0x0F, 0x38, 0x43, 0xE5, 0xEF, 0x98, 0xFC, 0x57, 0xD1, 0x2F, 0x70, 0xBC, 0xCB, 0x0C, 0xB5, 0x64, 0xB0, 0xAA, 0xBA, 0x96, 0x48, 0xD8, 0x9A, 0x99, 0x30, 0xCC, 0x11, 0x75, 0x4B, 0x8C, 0x62, 0xD7, 0x89, 0xC2, 0x07, 0xD2, 0x7D, 0x56, 0x9B, 0xA7, 0x06, 0x3D, 0x95, 0xEE, 0xE4, 0x8F, 0xFF, 0xD9, 0x37, 0x22, 0x44, 0xE0, 0xC8, 0xC9, 0x65, 0xC1, 0xAE, 0xC0, 0xB1, 0x14, 0xB6, 0x80, 0x53, 0x04, 0xF3, 0x25, 0x34, 0x6C, 0x4A, 0x94, 0x32, 0x68, 0xC6, 0x7F, 0xF5, 0x73, 0xE6, 0x3A, 0x6E, 0x83, 0xD6, 0xE8, 0x08, 0x42, 0x01, 0x00, 0x28, 0x61, 0x6F, 0x1D, 0x18, 0x6D, 0xE1, 0xA2, 0xEA, 0x1A, 0x29, 0x39, 0xB8, 0x46, 0xCA, 0xC7, 0xF8, 0xBF, 0xFD, 0x47, 0x91, 0x51, 0xBD, 0x3C, 0x10, 0x9C, 0x55, 0xB4, 0x5E, 0x09, 0x0E, 0xCF, 0x0D, 0xC5, 0x9F, 0x5C, 0x8B, 0x84, 0xE3, 0x05, 0x1E, 0x20, 0xFA, 0xF1, 0xDD, 0x7C, 0xD0, 0x31, 0xAB, 0x85, 0x6B, 0x49, 0x79, 0xA1, 0xD4, 0x2E, 0x1F, 0x40, 0x97, 0xFB, 0x90, 0x02, 0xC3, 0xF7, 0xF4, 0x8A, 0xF2, 0x9D, 0xEC, 0x81, 0x5F, 0x52, 0x72, 0xE7, 0xDA, 0x87, 0xB2, 0x23, 0x93, 0x21, 0xBB, 0x50, 0x67, 0x77, 0x6A, 0xED, 0x63, 0xA4, 0xB3, 0xBE]
enc=[0xCD, 0x8F, 0x25, 0x3D, 0xE1, 0x51, 0x4A]
j=0
i=0
for idx in range(len(enc)):
    i=(i+1)&0xff
    j=(j+sbox1[i])&0xff
    sbox1[i],sbox1[j]=sbox1[j],sbox1[i]
    k=(sbox1[i]+sbox1[j])&0xff
    enc[idx]^=sbox1[k]&0xff
print(''.join([chr(i) for i in enc]))
cipher=[0xF8, 0xD5, 0x62, 0xCF, 0x43, 0xBA, 0xC2, 0x23, 0x15, 0x4A, 0x51, 0x10, 0x27, 0x10, 0xB1, 0xCF, 0xC4, 0x09, 0xFE, 0xE3, 0x9F, 0x49, 0x87, 0xEA, 0x59, 0xC2, 0x07, 0x3B, 0xA9, 0x11, 0xC1, 0xBC, 0xFD, 0x4B, 0x57, 0xC4, 0x7E, 0xD0, 0xAA, 0x0A]
sbox2=[0x65, 0xC9, 0xDC, 0x3A, 0xCE, 0x59, 0xC0, 0x24, 0x48, 0xA0, 0x41, 0x62, 0x8F, 0x20, 0x26, 0xF8, 0x7C, 0xB4, 0xBA, 0x96, 0xE0, 0x5A, 0x2C, 0x19, 0x9D, 0x22, 0x93, 0xE4, 0x10, 0xE5, 0xC7, 0xBD, 0x3E, 0x76, 0xBE, 0xC6, 0x01, 0xFC, 0x86, 0x4F, 0xDD, 0xD9, 0xD4, 0x83, 0xD3, 0x77, 0x63, 0x97, 0xFD, 0x4A, 0xF7, 0xD5, 0xFA, 0x60, 0xF3, 0x6E, 0x32, 0x9E, 0x5C, 0x73, 0x61, 0xB5, 0x40, 0xDF, 0xE8, 0xF6, 0x80, 0x28, 0xCA, 0x45, 0xF0, 0xBC, 0xB8, 0xD7, 0x58, 0xCF, 0x9C, 0x69, 0x25, 0x52, 0x15, 0xCC, 0x70, 0x07, 0x7E, 0x06, 0x2E, 0x54, 0x1A, 0x35, 0x3B, 0x6F, 0x3C, 0x31, 0x7F, 0x1D, 0xF4, 0xE3, 0x82, 0xA7, 0x37, 0xF9, 0x50, 0x6D, 0x13, 0x46, 0x8D, 0x95, 0xAB, 0xB7, 0xAF, 0x72, 0xA8, 0xBB, 0x94, 0xAE, 0x5B, 0x67, 0xC1, 0xB3, 0xA4, 0x1C, 0x8C, 0x36, 0x14, 0xC4, 0xA5, 0xB2, 0x8A, 0xB0, 0x2D, 0x0B, 0x34, 0xCD, 0xA6, 0xFF, 0x21, 0x8B, 0xC8, 0x43, 0x00, 0x09, 0xF1, 0xD0, 0xB6, 0x23, 0x53, 0x84, 0x57, 0x64, 0xA2, 0x4B, 0x18, 0x0D, 0x5D, 0x78, 0x05, 0x02, 0x44, 0x92, 0x29, 0x7D, 0xFE, 0x08, 0x8E, 0xC3, 0x90, 0xE2, 0x1E, 0xE6, 0x81, 0x49, 0xE7, 0x6B, 0x12, 0x79, 0x0C, 0x33, 0xE1, 0x68, 0x27, 0xD1, 0x99, 0x03, 0x5F, 0xD2, 0xED, 0x0E, 0xB9, 0xCB, 0xEC, 0x4E, 0x56, 0x42, 0xDA, 0x87, 0xFB, 0x3D, 0xA1, 0x6A, 0x3F, 0x89, 0x0F, 0x51, 0x9B, 0x1B, 0x7A, 0x88, 0xEE, 0x30, 0x16, 0xEF, 0xC5, 0x9F, 0x74, 0x4C, 0xEB, 0x66, 0xB1, 0xDB, 0x6C, 0xD8, 0x47, 0x4D, 0xA9, 0x7B, 0x71, 0x2F, 0x1F, 0xAA, 0xD6, 0x2A, 0x2B, 0x91, 0x0A, 0x38, 0x85, 0xBF, 0xA3, 0x9A, 0x75, 0x55, 0x11, 0x98, 0x17, 0xC2, 0xF5, 0x39, 0xF2, 0xE9, 0xDE, 0x04, 0x5E, 0xEA, 0xAC, 0xAD]
i=0
j=0
for idx in range(len(cipher)):
    i=(i+1)&0xff
    j=(j+sbox2[i])&0xff
    sbox2[i],sbox2[j]=sbox2[j],sbox2[i]
    k=(sbox2[i]+sbox2[j])&0xff
    cipher[idx]=(cipher[idx]+sbox2[k])&0xff

print(''.join([chr(i) for i in cipher]))
```

`ecg4ab6`

`hgame{Y0u'r3_re4l1y_g3t_0Ut_of_th3_upX!}`

### delta_errors

差量补丁

[https://1k0ct.github.io/2024/04/29/Windows%E5%B7%AE%E5%BC%82%E5%8C%96%E8%A1%A5%E4%B8%81MSDelta%E4%B9%8B%E7%A0%94%E7%A9%B6/](https://1k0ct.github.io/2024/04/29/Windows%E5%B7%AE%E5%BC%82%E5%8C%96%E8%A1%A5%E4%B8%81MSDelta%E4%B9%8B%E7%A0%94%E7%A9%B6/)

我怀疑这个题目给的msdelta.dll有点问题

用我自己系统里的msdelta.dll就可以正常绕过效验，而用给的msdelta.dll一直失败

分析一下流程:

程序给了一个patch buf，但是哈希部分没了

真正效验部分是第二个appdeltaB

把打补丁后的输出作为key再xor密文

最后的输出只与补丁有关

我们只用patch补丁绕过效验即可

```
patch=bytes([0x50, 0x41, 0x33, 0x30, 0x30, 0x0B, 0xD0, 0x45, 0x74, 0x6C, 0xDB, 0x01, 0x18, 0x23, 0xC8, 0x81, 0x03, 0x80, 0x42, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x01, 0x7A, 0x00, 0x51, 0xB5, 0x5E, 0x73, 0x7A, 0x8D, 0xF1, 0x30, 0xAD, 0xD3, 0xA2, 0x69, 0x1E, 0x16, 0x8D, 0x9B, 0xE5, 0x6F, 0x4A, 0x2F, 0x0F, 0x53, 0x06, 0xF5, 0x1B, 0x30, 0xC3, 0x73, 0x16, 0x0D])
buf = b"A"*36
newheader = patch[:14] + b'\xc8\x11\x02' + patch[16+ 0x14:]
#print(list(newheader))
with open("newheader2.bin","wb") as f:
    f.write(newheader)
#python .\delta_patch.py -i input -o output newheader2.bin
```

拿到恢复后的key

```
b"Seven says you're right!!!!\x00"
```

```
a = b"Seven says you're right!!!!\x00"

cipher=[0x3B, 0x02, 0x17, 0x08, 0x0B, 0x5B, 0x4A, 0x52, 0x4D, 0x11, 0x11, 0x4B, 0x5C, 0x43, 0x0A, 0x13, 0x54, 0x12, 0x46, 0x44, 0x53, 0x59, 0x41, 0x11, 0x0C, 0x18, 0x17, 0x37, 0x30, 0x48, 0x15, 0x07, 0x5A, 0x46, 0x15, 0x54, 0x1B, 0x10, 0x43, 0x40, 0x5F, 0x45, 0x5A]

for i in range(len(cipher)):
    print(chr(a[i%len(a)]^cipher[i]),end='')
```

### 尊嘟假嘟

libcheck.so在我的root安卓设备里加载不上，一点就闪退，难绷

分析一下这个题目的解题流程和思路:

1. 用户分别每次点击0.O和o.0 ，连接形成了一个字符串，传入toast.setText函数
2. setText首先是call了个DexMethod，这个dex在资源里被加密了

这里我直接hook了file.delete()方法，让资源解密后不被删除然后拉到本机电脑继续分析

```js
Java.perform(function () {
    // Hook 'DexCall.callDexMethod'
    var DexCall = Java.use('com.nobody.zunjia.DexCall');
    
    // Hook 'java.io.File' to control file behavior
    var File = Java.use('java.io.File');
    
    // Hook the 'exists' method of 'File'
    File.exists.implementation = function () {
        // Log for debugging purposes
        console.log('Hooked File.exists() method');
        
        // Always return true, simulating that the file always exists
        return true;
    };
    
    // Hook the 'delete' method of 'File' to prevent actual file deletion
    File.delete.implementation = function () {
        console.log('Hooked File.delete() method - preventing deletion');
        return true; // Prevent the actual file from being deleted
    };
    
    // Hook the method call in 'DexCall'
    DexCall.callDexMethod.implementation = function (context, dexFileName, className, methodName, input) {
        console.log('Hooked callDexMethod');
        
        // Call the original method, but let the `File.exists()` be controlled by Frida
        return this.callDexMethod(context, dexFileName, className, methodName, input);
    };
    
    console.log('Frida hook setup complete!');
});
```

3. 分析dex发现是一个魔改的base64
4. 然后进入libcheck.so里进行check，发现其实并没有check，只是进行了一个rc4后，又进行了一个之前的魔改base64然后输出到log中
5. 由于我的设备不能动调libcheck.so，脑洞了一下，key应该是第3步返回后的加密的值，验证结果是正确的，然后就开始爆破

```python
import base64

class Zundujiadu:
    CUSTOM_ALPHABET = "3GHIJKLMNOPQRSTUb=cdefghijklmnopWXYZ/12+406789VaqrstuvwxyzABCDEF5"
    STANDARD_ALPHABET = "ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789+/="
    DECODE_TABLE = {char: index for index, char in enumerate(CUSTOM_ALPHABET)}

    def encode(self, data):
        if data is None:
            return None
        # XOR each byte with its index
        data = bytes([byte ^ index for index, byte in enumerate(data)])
        # Standard Base64 encoding
        encoded = base64.b64encode(data).decode('utf-8')
        # Replace standard Base64 characters with custom alphabet
        translation_table = str.maketrans(self.STANDARD_ALPHABET, self.CUSTOM_ALPHABET)
        return encoded.translate(translation_table)

    def decode(self, data):
        if data is None:
            return None
        # Replace custom alphabet characters with standard Base64 characters
        translation_table = str.maketrans(self.CUSTOM_ALPHABET, self.STANDARD_ALPHABET)
        data = data.translate(translation_table)
        # Standard Base64 decoding
        decoded = base64.b64decode(data)
        # XOR each byte with its index
        return bytes([byte ^ index for index, byte in enumerate(decoded)])

# Example usage
zj = Zundujiadu()
#vpEoBVwC1PHzMxuoJNp4VfP7ctjSgvZ2YCfDhKxcAajt9qCdMAKNE/Et0333
#U4BGQ7rzHNT+9bPAtGzl3BpW=/fAHyZEqjyqY=IjXiDd1S+EHtgiCwFdC333
key1="o.0" 
key2="0.o"
key=zj.encode(bytes(key1.encode()))
sbox=[0]*256
keybox=[0]*256
for i in range(256):
    sbox[i] = i
for i in range(256):
    keybox[i] = ord(key[i%len(key)])
j=0
for i in range(256):
    j = (j + sbox[i] + keybox[i]) % 256
    sbox[i], sbox[j] = sbox[j], sbox[i]

#print(sbox)
cipher=[0x7A, 0xC7, 0xC7, 0x94, 0x51, 0x82, 0xF5, 0x99, 0x0C, 0x30, 0xC8, 0xCD, 0x97, 0xFE, 0x3D, 0xD2, 0xAE, 0x0E, 0xBA, 0x83, 0x59, 0x87, 0xBB, 0xC6, 0x35, 0xE1, 0x8C, 0x59, 0xEF, 0xAD, 0xFA, 0x94, 0x74, 0xD3, 0x42, 0x27, 0x98, 0x77, 0x54, 0x3B, 0x46, 0x5E, 0x95]

i=0
j=0
for idx in range(len(cipher)):
    i=(i+1)%256
    j=(j+sbox[i])%256
    sbox[i],sbox[j]=sbox[j],sbox[i]
    cipher[idx]=cipher[idx]^sbox[(sbox[i]+sbox[j])%256]
print(cipher)
print(zj.encode(bytes(cipher)))
print(len(zj.encode(bytes(cipher))))

def try_key(key):
    zj = Zundujiadu()
    #print(f"Trying key:{key}")
    key=zj.encode(bytes(key.encode()))
    sbox=[0]*256
    keybox=[0]*256
    for i in range(256):
        sbox[i] = i
    for i in range(256):
        keybox[i] = ord(key[i%len(key)])
    j=0
    for i in range(256):
        j = (j + sbox[i] + keybox[i]) % 256
        sbox[i], sbox[j] = sbox[j], sbox[i]

    #print(sbox)
    cipher=[0x7A, 0xC7, 0xC7, 0x94, 0x51, 0x82, 0xF5, 0x99, 0x0C, 0x30, 0xC8, 0xCD, 0x97, 0xFE, 0x3D, 0xD2, 0xAE, 0x0E, 0xBA, 0x83, 0x59, 0x87, 0xBB, 0xC6, 0x35, 0xE1, 0x8C, 0x59, 0xEF, 0xAD, 0xFA, 0x94, 0x74, 0xD3, 0x42, 0x27, 0x98, 0x77, 0x54, 0x3B, 0x46, 0x5E, 0x95]

    i=0
    j=0
    for idx in range(len(cipher)):
        i=(i+1)%256
        j=(j+sbox[i])%256
        sbox[i],sbox[j]=sbox[j],sbox[i]
        cipher[idx]=cipher[idx]^sbox[(sbox[i]+sbox[j])%256]
    
    try:
        print(''.join([chr(i) for i in cipher]))
    except Exception as e:
        pass

#maxlen(36) 12次
from collections import deque

queue=deque([key1,key2])
visited=set((key1,key2))

while queue:
    curr=queue.popleft()
    if len(curr)>36:
        continue
    try_key(curr)
    for next_key in [key1,key2]:
        new_key=curr+next_key
        if len(new_key)<=36 and new_key not in visited:
            visited.add(new_key)
            queue.append(new_key)
```

## Week2

### signin

太阴间了这道题

下硬件断点会导致tea加密的key是错误的

下软件断点会导致tea sum的计算方式是错误的

先只下软件断点拿正确的key

再只下硬件断点还原正确的算法，并验证结果

```c
#include<stdio.h>
#include<stdint.h>
 
void encipher(unsigned int num_rounds, uint32_t v[9], uint32_t const key[4]){
	unsigned int i;
	uint32_t v0=0,v9=v[8],sum=0,delta=0,v1=0,tmp=0,v2=0;
    uint32_t rcon[4]={
        0, 0, 0, 0
    };
    sum = 0;
    do
    {
      sum += rcon[num_rounds % 4];
      tmp = (sum >> 2) & 3;

      for ( i = 0; i < 8; ++i )
      {

        v1 = v[i + 1];
        v0 = (((v9 ^ key[tmp^i&3]) + (v1 ^ sum)) ^ (((16 * v9) ^ (v1 >> 3))
                                                                             + ((4 * v1) ^ (v9 >> 5))))
           + v[i];
        v[i] = v0;
        v9 = v0;
      }
      v2 = (((v9 ^  key[tmp^i&3]) + (v[0] ^ sum)) ^ (((16 * v9) ^ (v[0] >> 3))
                                                                           + ((4 * v[0]) ^ (v9 >> 5))))
         + v[8];
        v[8] = v2;
        v9 = v2;
        for(int i=0;i<9;++i)
        {
            printf("%08x ",v[i]);
            }
        printf("\n");
        num_rounds--;
    } while ( num_rounds!=-1 );

}
 
void decipher(unsigned int num_rounds, uint32_t v[9], uint32_t const key[4]) {
    unsigned int i;
    uint32_t v9=0, sum=0, v2=0, tmp=0;
    uint32_t original_rounds = num_rounds;
    
    // uint32_t rcon[4]={
    //     0x4BD7F80D, 0x00007FF7, 0x4BD7F83E, 0x00007FF7
    // };
    uint32_t rcon[4]={
        0, 0, 0, 0
    };
    // Calculate initial sum
    sum = 0;
    do {
        sum += rcon[original_rounds%4];
    } while(--original_rounds);
    
    num_rounds = 11;
    do {
        tmp = (sum >> 2) & 3;
        
        // v8 <- v0,v7
        v9 = v[7];
        v2 = v[8]-(((v9 ^ key[tmp^i&3]) + (v[0] ^ sum)) ^ 
             (((16 * v9) ^ (v[0] >> 3))+ ((4 * v[0]) ^ (v9 >> 5))));
        v[8] = v2;
        
        // v0 <- v1+v8
        int i=7;
        do{
            uint32_t v1 = v[i+1];
            if (i == 0) {
                v9 = v[8];  // 当 i=0 时，使用 v[8] 来避免越界
            } else {
                v9 = v[i-1];
            }
            v[i] = v[i] - (((v9 ^ key[tmp ^ i & 3]) + (v1 ^ sum)) ^ 
                           (((16 * v9) ^ (v1 >> 3)) + ((4 * v1) ^ (v9 >> 5))));
            i--;
        }while(i!=-1);
        
        sum -= rcon[num_rounds % 4];
        
        // Print current state
        unsigned int j;
        for(j = 0; j < 9; ++j) {
            printf("%08x ", v[j]);
        }
        printf("\n");
    } while (--num_rounds);
}
int main(){
	uint32_t v[9]={ 0x3050EA23, 0x47514C00, 0x2B769CEE, 0x1794E6D5, 0xB3E42BED, 0x61D536CB, 0x7CA0C2C0, 0x5ED767FE, 0xC579E0AF};
	uint32_t const k[4]={  0x97A25FB5, 0xE1756DBA, 0xA143464A, 0x5A8F284F};
	unsigned int r=11;				

    uint32_t inp[9]={0x61616161,0x61616161,0x61616161,0x61616161,0x61616161,0x61616161,0x61616161,0x61616161,0x61616161};
    uint32_t cip[9]={   0x3050EA23, 0x47514C00, 0x2B769CEE, 0x1794E6D5, 0xB3E42BED, 0x61D536CB, 0x7CA0C2C0, 0x5ED767FE, 0xC579E0AF};
    uint32_t res[9]={0x051927C3, 0x5FACCDE4, 0xA918FC65, 0xA97B04B2, 0x56D6F14F, 0x2709CD55, 0x8051EB07, 0x87E53DC2, 
        0x45D7F1E7};
    encipher(r,inp,k);
    decipher(r,cip,k);
    for(int i=0;i<9;++i)
    {
        printf("0x%08x,",inp[i]);
    }
    printf("\n");
    uint8_t* cip2=(uint8_t*)cip;
    for(int i=0;i<36;++i)
    {
        printf("%c",cip2[i]);
    }

	return 0;
}
#3fe4722c-1dbf-43b7-8659-c1c4a0e42e4d
```

### nopd

两个进程间通信实现了一个加密的过程，分析game没分析出来什么东西，看launcher

```c
__int64 __fastcall shenmihanshu(unsigned int pid)
{
    unsigned int v1; // ebx
    char i; // bl
    int v3; // ebx
    __int64 v4; // rax
    __int64 v5; // r15
    char v6; // dl
    __int64 v7; // r15
    __int64 (__fastcall *v8)(); // rax
    unsigned int pida; // [rsp+4h] [rbp-64h]
    __int64 v11; // [rsp+18h] [rbp-50h] BYREF
    char v12[4]; // [rsp+22h] [rbp-46h] BYREF
    unsigned __int8 v13; // [rsp+26h] [rbp-42h]
    unsigned __int8 v14; // [rsp+27h] [rbp-41h]
    unsigned __int64 v15; // [rsp+28h] [rbp-40h]

    v1 = pid;
    v15 = __readfsqword(0x28u);
    waitpid(pid, 0LL, 0);
    ptrace(PTRACE_SETOPTIONS, pid, 0LL, 0x100000LL);
    while ( ptrace(PTRACE_SYSCALL, v1, 0LL, 0LL) >= 0 )
    {
        waitpid(v1, 0LL, 0);
        ptrace(PTRACE_GETREGS, v1, 0LL, &qword_5616E26D95E0);
        sub_5616E26D534B(PTRACE_PEEKTEXT, v1, (__int64)byte_5616E26D95C0, *(&qword_5616E26D95E0 + 16) - 8, 6uLL);
        if ( (byte_5616E26D95C0[0] & 0xFC) != 72
        || byte_5616E26D95C0[1] != 15
        || byte_5616E26D95C2 != 31
        || (byte_5616E26D95C3 & 0xC7) != 68
        || byte_5616E26D95C5 != 127 )
        {
            v11 = *(&qword_5616E26D95E0 + 16);
            goto LABEL_41;
        }
        memset(byte_5616E26D9540, 0, sizeof(byte_5616E26D9540));
        qword_5616E26D9520 = 0LL;
        pida = v1;
        for ( i = 0; ; i = 1 )
        {
            v5 = qword_5616E26D9520;
            sub_5616E26D534B(PTRACE_PEEKTEXT, pida, (__int64)v12, qword_5616E26D9660 + 6 * qword_5616E26D9520, 6uLL);
            if ( (v12[0] & 0xFC) != 72 || v12[1] != 15 || v12[2] != 31 || (v12[3] & 0xC7) != 68 || v14 == 126 )
                break;
            v3 = (v14 - 1) % 128;
            if ( v13 > 0x3Fu )
            {
                sub_5616E26D5464(
                    pida,
                    &v11,
                    qword_5616E26D9678 + ((8 * (v13 & 7 | (unsigned __int8)(8 * (v12[0] & 1)))) & 0x78),
                    8LL);
                qword_5616E26D9120[(unsigned __int8)v3] = v11;
            }
            else
            {
                switch ( v13 & 7 | (unsigned __int8)(8 * (v12[0] & 1)) )
                {
                    case 0:
                        v4 = qword_5616E26D9658;
                        break;
                    case 1:
                        v4 = qword_5616E26D9638;
                        break;
                    case 2:
                        v4 = qword_5616E26D9640;
                        break;
                    case 3:
                        v4 = qword_5616E26D9608;
                        break;
                    case 4:
                        v4 = qword_5616E26D9678;
                        break;
                    case 5:
                        v4 = qword_5616E26D9600;
                        break;
                    case 6:
                        v4 = qword_5616E26D9648;
                        break;
                    case 7:
                        v4 = qword_5616E26D9650;
                        break;
                    case 8:
                        v4 = qword_5616E26D9628;
                        break;
                    case 9:
                        v4 = qword_5616E26D9620;
                        break;
                    case 10:
                        v4 = qword_5616E26D9618;
                        break;
                    case 11:
                        v4 = qword_5616E26D9610;
                        break;
                    case 12:
                        v4 = qword_5616E26D95F8;
                        break;
                    case 13:
                        v4 = qword_5616E26D95F0;
                        break;
                    case 14:
                        v4 = qword_5616E26D95E8;
                        break;
          case 15:
            v4 = qword_5616E26D95E0;
            break;
        }
        qword_5616E26D9120[(unsigned __int8)v3] = v4;
      }
      byte_5616E26D9540[(unsigned __int8)v3] = 1;
      qword_5616E26D9520 = v5 + 1;
    }
    v6 = i;
    v1 = pida;
    v11 = *(&qword_5616E26D95E0 + 16);
    if ( v6 )
    {
      v7 = -38LL;
      if ( byte_5616E26D9540[0] )
      {
        if ( qword_5616E26D9120[0] <= 9uLL )
        {
          v8 = funcs_1DBD[qword_5616E26D9120[0]];
          if ( v8 )
            v7 = ((int (__fastcall *)(_QWORD, __int64 *, char *, __int64 *))v8)(
                   pida,
                   qword_5616E26D9120,
                   byte_5616E26D9540,
                   &v11);
        }
        *(&qword_5616E26D95E0 + 15) = -1LL;
        ptrace(PTRACE_SETREGS, pida, 0LL, &qword_5616E26D95E0);
      }
      ptrace(PTRACE_SYSCALL, pida, 0LL, 0LL);
      waitpid(pida, 0LL, 0);
      ptrace(PTRACE_GETREGS, pida, 0LL, qword_5616E26D9040);
      qword_5616E26D9040[10] = v7;
      qword_5616E26D9040[16] = v11;
      ptrace(PTRACE_SETREGS, pida, 0LL, qword_5616E26D9040);
    }
    else
    {
LABEL_41:
      ptrace(PTRACE_SYSCALL, v1, 0LL, 0LL);
      waitpid(v1, 0LL, 0);
    }
  }
  return 0LL;
}
```

![[2dd826b40d97cde1de4e370e1520a44a_MD5.png]]

![[7373b27cf16d7a31b48a18a26ba4ac65_MD5.png]]

![[5a8162e15f075159ac21bf86fa4e4c22_MD5.png]]

![[8053c592efe00440c635191a458f2699_MD5.png]]

```
struct user_regs_struct {
    unsigned long    r15;
    unsigned long    r14;
    unsigned long    r13;
    unsigned long    r12;
    unsigned long    bp;
    unsigned long    bx;
    unsigned long    r11;
    unsigned long    r10;
    unsigned long    r9;
    unsigned long    r8;
    unsigned long    ax;
    unsigned long    cx;
    unsigned long    dx;
    unsigned long    si;
    unsigned long    di;
    unsigned long    orig_ax;
    unsigned long    ip;
    unsigned long    cs;
    unsigned long    flags;
    unsigned long    sp;
    unsigned long    ss;
    unsigned long    fs_base;
    unsigned long    gs_base;
    unsigned long    ds;
    unsigned long    es;
    unsigned long    fs;
    unsigned long    gs;
};
```

导入结构体

![[58abe1d3ff811d7a229739faa93c1fe3_MD5.png]]

奇怪tea常量

![[4d4f4dcc7cd448474a7e17ba32440e7c_MD5.png]]

奇怪算法

然后又是一个函数跳转表

```
0x55576871E329: ret0
0x55576871E73E: putsdata
0x55576871E7DD: output
0x55576871E89F: enc1
0x55576871E9C1: enc2
0x55576871E54E: load_cipher
0x55576871E64A: add
0x55576871E333: ret_delta
0x55576871E33D: rcx_add0x80
```

```
import idc
rax=idc.get_reg_value('rax')
offset=rax&0xfff
if offset==0x329:#set eax=0  没用的call
    print('',end='')
elif offset==0x73E:
    print("called:putsdata")
elif offset==0x7DD:
    print("called:output")
elif offset==89F:
    print("called:enc1")
elif offset==0x9C1:
    print("called:enc2")
elif offset==0x54E:
    print("called:load_cipher")
elif offset==0x64A:
    print("called:add")
elif offset==0x333:
    print("called:ret_delta")
elif offset==0x33D:
    print("called:rcx_add0x80")
```

先trace一下 看看怎么个事

```
called:output
called:getinput
called:load_cipher
called:enc1
called:enc2
called:enc1
called:enc2
called:enc1
called:enc2
called:enc1
called:enc2
called:enc1
called:enc2
called:enc1
called:enc2
called:enc1
called:enc2
called:enc1
called:enc2
called:enc1
called:enc2
called:enc1
called:enc2
called:enc1
called:enc2
called:enc1
called:enc2
called:enc1
called:enc2
called:enc1
called:enc2
called:enc1
called:enc2
called:enc1
called:enc2
called:enc1
called:enc2
called:enc1
called:enc2
called:enc1
called:enc2
called:enc1
called:enc2
called:add
called:ret_delta
called:rcx_add0x80
called:cmp_res
```

经过了20轮的enc1和enc2

现在详细分析每个函数的功能

![[62c50f699f344f583fcac9869d1c435c_MD5.png]]

```
发现load了一个struct #应该是key  然后每个byte都-4
aExpand32ByteK  db 'expand 32-byte k',0 
[+] Dump 0x7FFD69C71B40 - 0x7FFD69C71B4F (16 bytes) :
[0x65, 0x78, 0x70, 0x61, 0x6E, 0x64, 0x20, 0x33, 0x32, 0x2D, 0x62, 0x79, 0x74, 0x65, 0x20, 0x6B]
putsdata
[0x65, 0x78, 0x70, 0x61, 0x6E, 0x64, 0x20, 0x33, 0x32, 0x2D, 0x62, 0x79, 0x74, 0x65, 0x20, 0x6B, 0x49, 0x74, 0x27, 0x73, 0x20, 0x61, 0x6C, 0x6C, 0x20, 0x77, 0x72, 0x69, 0x74, 0x74, 0x65, 0x6E, 0x20, 0x69, 0x6E, 0x20, 0x74, 0x68, 0x65, 0x20, 0x42, 0x6F, 0x6F, 0x6B, 0x20, 0x6F, 0x66, 0x20, 0x00, 0x00, 0x00, 0x00, 0x57, 0x68, 0x61, 0x74, 0x27, 0x73, 0x20, 0x79, 0x6F, 0x75, 0x72, 0x20]
```

这里盲猜是load_key了

enc1是一堆simd整形指令，用C还原

enc2含有shuffle指令

![[aa5f56d9b1d26d3b1641dcf7062b6fa6_MD5.png]]

0x39 : 00 11 10 01

0x4e: 01 00 11 10

0x93: 10 01 00 11

main函数有个ptrace反调试，nop掉后还是调不了，摆了摆了

20轮enc都能分析清楚，这里把key给算好了然后数据传来传去的，整不明白，input到底是怎么被game给处理的

用strace了一下，看不太懂，又不会调试game，摆了摆了等着看wp了

```c
#include <stdio.h>
#include <stdint.h>

int enc1(uint8_t a[], uint8_t b[],uint8_t c[],uint8_t d[])
{
    uint32_t* a1=(uint32_t*)a;
    uint32_t* b1=(uint32_t*)b;
    uint32_t* c1=(uint32_t*)c;
    uint32_t* d1=(uint32_t*)d;
    uint8_t v4[16];uint32_t* v4_1=(uint32_t*)v4;
    uint8_t v5[16];uint32_t* v5_1=(uint32_t*)v5;
    uint8_t v7[16];uint32_t* v7_1=(uint32_t*)v7;
    uint8_t v8[16];uint32_t* v8_1=(uint32_t*)v8;
    uint8_t v10[16];uint32_t* v10_1=(uint32_t*)v10;
    uint8_t v11[16];uint32_t* v11_1=(uint32_t*)v11;

    //29:v4 = _mm_load_si128(&b);
    for(int i=0;i<16;i++)
    {
        v4[i]=b[i];
    }
    //  v5 = _mm_add_epi32(v4, key);
    for (int i = 0; i <4 ;i++)
    {
        a1[i] = a1[i] + b1[i];
        v5_1[i] = a1[i];
    }
    // for(int i=0;i<4;i++){printf("%8X ",a1[i]);}printf("\n");

    //v6 = _mm_xor_si128(v5, d);
    for(int i=0;i<16;i++)
    {
        a[i]^=d[i];
    }
    //for(int i=0;i<4;i++){printf("%8X ",a1[i]);}printf("\n");



    //32: v7 = _mm_or_si128(_mm_slli_epi32(v6, 0x10u), _mm_srli_epi32(v6, 0x10u));
    for(int i=0;i<4;i++)
    {
        uint32_t tmp;
        a1[i]=a1[i]<<16|a1[i]>>16;
        v7_1[i]=a1[i];

    }
    // for(int i=0;i<4;i++){printf("%8X ",a1[i]);}printf("\n");


    //v8 = _mm_add_epi32(v7, c);
    for(int i=0;i<4;i++)
    {
        a1[i]+=c1[i];
        v8_1[i]=a1[i];
    }
    //for(int i=0;i<4;i++){printf("%8X ",a1[i]);}printf("\n");


    //v9 = _mm_xor_si128(v4, v8);
    for(int i=0;i<16;i++)
    {
        a[i]^=v4[i];
    }
    //for(int i=0;i<4;i++){printf("%8X ",a1[i]);}printf("\n");


    //v10 = _mm_or_si128(_mm_slli_epi32(v9, 0xCu), _mm_srli_epi32(v9, 0x14u));
    for(int i=0;i<4;i++)
    {

        a1[i]=a1[i]<<0xc|a1[i]>>0x14;
        v10_1[i]=a1[i];
    }
    //for(int i=0;i<4;i++){printf("%8X ",a1[i]);}printf("\n");


    //v11 = _mm_add_epi32(v5, v10);
    for(int i=0;i<4;i++)
    {
        a1[i]+=v5_1[i];
        v11_1[i]=a1[i];
    }
    // for(int i=0;i<4;i++){printf("%8X ",a1[i]);}printf("\n");


    //v12 = _mm_xor_si128(v7, v11);
    for(int i=0;i<16;i++)
    {
        a[i]^=v7[i];
    }
    //for(int i=0;i<4;i++){printf("%8X ",a1[i]);}printf("\n");


    //v13 = _mm_or_si128(_mm_slli_epi32(v12, 8u), _mm_srli_epi32(v12, 0x18u));
    for(int i=0;i<4;i++)
    {
        a1[i]=a1[i]<<0x8|a1[i]>>0x18;
        d1[i]=a1[i];
    }
    //for(int i=0;i<4;i++){printf("%8X ",a1[i]);}printf("\n");


    //v14 = _mm_add_epi32(v8, v13);
    for(int i=0;i<4;i++)
    {
        a1[i]+=v8_1[i];
        c1[i]=a1[i];
    }
    //for(int i=0;i<4;i++){printf("%8X ",a1[i]);}printf("\n");


    //v15 = _mm_xor_si128(v10, v14);
    for(int i=0;i<16;i++)
    {
        a[i]^=v10[i];
    }
    //for(int i=0;i<4;i++){printf("%8X ",a1[i]);}printf("\n");


    //b = _mm_or_si128(_mm_slli_epi32(v15, 7u), _mm_srli_epi32(v15, 0x19u));
    for(int i=0;i<4;i++)
    {
        a1[i]=a1[i]<<7|a1[i]>>0x19;
        b1[i]=a1[i];
    }
    // for(int i=0;i<4;i++){printf("%8X ",a1[i]);}printf("\n");


    //key=v11
    for(int i=0;i<16;i++)
    {
        a[i]=v11[i];
    }
    // for(int i=0;i<4;i++){printf("%8X ",a1[i]);}printf("\n");
    // for(int i=0;i<4;i++){printf("%8X ",b1[i]);}printf("\n");
    // for(int i=0;i<4;i++){printf("%8X ",c1[i]);}printf("\n");
    // for(int i=0;i<4;i++){printf("%8X ",d1[i]);}printf("\n");
    
    return 0;
}

int enc2(uint8_t a[],uint8_t b[],uint8_t c[],uint8_t d[],int flags)
{
    uint32_t* a1=(uint32_t*)a;
    uint32_t* b1=(uint32_t*)b;
    uint32_t* c1=(uint32_t*)c;
    uint32_t* d1=(uint32_t*)d;
    uint32_t tmp[4];
    // for(int i=0;i<16;i++){printf("%02x ",b[i]);}printf("\n");
    // for(int i=0;i<16;i++){printf("%02X ",c[i]);}printf("\n");
    // for(int i=0;i<16;i++){printf("%02X ",d[i]);}printf("\n");
    if(flags)
    {
        for(int i=0;i<4;i++)
        {
            tmp[i]=b1[i];
        }
        b1[3]=tmp[0];b1[2]=tmp[3];b1[1]=tmp[2];b1[0]=tmp[1];
        for(int i=0;i<4;i++)
        {
            tmp[i]=c1[i];
        }
        c1[3]=tmp[1];c1[2]=tmp[0];c1[1]=tmp[3];c1[0]=tmp[2];
        for(int i=0;i<4;i++)
        {
            tmp[i]=d1[i];
        }
        d1[3]=tmp[2],d1[2]=tmp[1];d1[1]=tmp[0];d1[0]=tmp[3];
    }
    else
    {
        for(int i=0;i<4;i++)
        {
            tmp[i]=d1[i];
        }
        d1[3]=tmp[0];d1[2]=tmp[3];d1[1]=tmp[2];d1[0]=tmp[1];
        for(int i=0;i<4;i++)
        {
            tmp[i]=c1[i];
        }
        c1[3]=tmp[1];c1[2]=tmp[0];c1[1]=tmp[3];c1[0]=tmp[2];
        for(int i=0;i<4;i++)
        {
            tmp[i]=b1[i];
        }
        b1[3]=tmp[2],b1[2]=tmp[1];b1[1]=tmp[0];b1[0]=tmp[3];
    }
    for(int i=0;i<4;i++){printf("%8X ",b1[i]);}printf("\n");
    for(int i=0;i<4;i++){printf("%8X ",c1[i]);}printf("\n");
    for(int i=0;i<4;i++){printf("%8X ",d1[i]);}printf("\n");

    // for(int i=0;i<16;i++){printf("%02x ",a[i]);}printf("\n");
    // for(int i=0;i<16;i++){printf("%02x ",b[i]);}printf("\n");
    // for(int i=0;i<16;i++){printf("%02X ",c[i]);}printf("\n");
    // for(int i=0;i<16;i++){printf("%02X ",d[i]);}printf("\n");
}
int add(uint8_t a[],uint8_t b[])
{
    uint32_t* a1=(uint32_t*)a;
    uint32_t* b1=(uint32_t*)b;
    for(int i=0;i<16;i++)
    {
        a1[i]+=b1[i];
    }
    return 0;
}
int main()
{
    uint8_t addarray[64]={0x65, 0x78, 0x70, 0x61, 0x6E, 0x64, 0x20, 0x33, 0x32, 0x2D, 0x62, 0x79, 0x74, 0x65, 0x20, 0x6B, 0x49, 0x74, 0x27, 0x73, 0x20, 0x61, 0x6C, 0x6C, 0x20, 0x77, 0x72, 0x69, 0x74, 0x74, 0x65, 0x6E, 0x20, 0x69, 0x6E, 0x20, 0x74, 0x68, 0x65, 0x20, 0x42, 0x6F, 0x6F, 0x6B, 0x20, 0x6F, 0x66, 0x20, 0x00, 0x00, 0x00, 0x00, 0x57, 0x68, 0x61, 0x74, 0x27, 0x73, 0x20, 0x79, 0x6F, 0x75, 0x72,0x20};
    uint8_t key[16] = {0x65, 0x78, 0x70, 0x61, 0x6E, 0x64, 0x20, 0x33, 0x32, 0x2D, 0x62, 0x79, 0x74, 0x65, 0x20, 0x6B};
    uint8_t b[16]={0x49, 0x74, 0x27, 0x73, 0x20, 0x61, 0x6C, 0x6C, 0x20, 0x77, 0x72, 0x69, 0x74, 0x74, 0x65, 0x6E};
    uint8_t c[]={0x20, 0x69, 0x6E, 0x20, 0x74, 0x68, 0x65, 0x20, 0x42, 0x6F, 0x6F, 0x6B, 0x20, 0x6F, 0x66, 0x20};
    uint8_t d[]={0x00, 0x00, 0x00, 0x00, 0x57, 0x68, 0x61, 0x74, 0x27, 0x73, 0x20, 0x79, 0x6F, 0x75, 0x72, 0x20};
    for(int i=0;i<20;i++)
    {
        printf("round %d\n",i);
        int flags=(i+1)%2;
        enc1(key,b,c,d);
        enc2(key,b,c,d,flags);
    }
    for(int i=0;i<16;i++){printf("%02x ",key[i]);}printf("\n");
    for(int i=0;i<16;i++){printf("%02x ",b[i]);}printf("\n");
    for(int i=0;i<16;i++){printf("%02X ",c[i]);}printf("\n");
    for(int i=0;i<16;i++){printf("%02X ",d[i]);}printf("\n");
    uint8_t new[64]={0};
    for(int i=0;i<64;i++)
    {
        if(i<16)
            new[i]=key[i];
        else if(i<32)
            new[i]=b[i%16];
        else if(i<48)
            new[i]=c[i%16];
        else
            new[i]=d[i%16];
    }
    add(new,addarray);
    for(int i=0;i<64;i++)
    {
        printf("%02x ",new[i]);
        if(i%16==15)
            printf("\n");
    }

    return 0;
}
```

但是观察加密特征发现是单字节加密的，其实可以进行取巧爆破

用LD_PRELOAD_HOOK memcmp函数

```
#define _GNU_SOURCE
#include <stdio.h>
#include <dlfcn.h>
#include <string.h>
#include <stdint.h>

// 定义一个函数指针，用于保存原始的 `memcmp` 函数地址
typedef int (*orig_memcmp_t)(const void *, const void *, size_t);

// 辅助函数：以 hex 形式打印数据
void print_hex(const void *data, size_t len) {
    const uint8_t *p = (const uint8_t *)data;
    for (size_t i = 0; i < len; i++) {
        printf("%02X ", p[i]);  // 以 "%02X" 格式打印，每字节两位
    }
}

// 自定义 `memcmp` hook
int memcmp(const void *s1, const void *s2, size_t n) {
    static orig_memcmp_t orig_memcmp = NULL;

    // 通过 dlsym 获取原始 `memcmp` 地址
    if (!orig_memcmp) {
        orig_memcmp = (orig_memcmp_t)dlsym(RTLD_NEXT, "memcmp");
    }

    // // 打印 `memcmp` 调用信息
    // printf("[HOOK] memcmp called: n = %zu\n", n);

    // print_hex(s1, n);
    // printf("\n");

    //printf("[HOOK] s2: ");
    print_hex(s2, n);
    printf("\n");

    // 调用原始 `memcmp` 函数
    return orig_memcmp(s1, s2, n);
}
```

编译成静态库

```
gcc -shared -fPIC -o hook.so hook.c -ldl
LD_PRELOAD=./hook.so ./launcher game
```

```
import os
import pty
import subprocess

def try_inp(inp):
    # 创建伪终端
    master, slave = pty.openpty()
    
    # 启动子进程，设置环境变量和终端
    proc = subprocess.Popen(
        ['./launcher', './game'],
        env={**os.environ, 'LD_PRELOAD': './hook.so'},
        stdin=subprocess.PIPE,
        stdout=slave,
        stderr=subprocess.PIPE,
        close_fds=True,
    )
    os.close(slave)  # 关闭不再需要的slave端
    
    # 读取直到出现'?'提示符
    prompt = b''
    while True:
        try:
            char = os.read(master, 1)
            #print(char)
        except OSError:
            break
        if not char:
            break
        prompt += char
        if b'?' in prompt:
            break
    
    # 发送输入'hgame'后换行
    proc.stdin.write(inp.encode()+b'\n')
    proc.stdin.flush()
    
    # 读取十六进制字符串行
    char = os.read(master, 3)

    hex_output = b''
    while True:
        #print(char)
        char =os.read(master,1)
        if char in (b'\n', b'') or len(hex_output) >= 1024:
            break
        hex_output += char
    #print(hex_output)
    # 清理资源
    proc.stdin.close()
    os.close(master)
    proc.terminate()
    
    return hex_output.decode()
    #rprint(hex_output.decode().strip())

from string import printable 
if __name__ == "__main__":
    inp='hgame{'
    cipher=[0x64, 0x6A, 0x50, 0x17, 0x81, 0x7D, 0x6F, 0x1A, 0x87, 0xB1, 0xA4, 0x00, 0x09, 0x03, 0xF8, 0x8D, 0xF8, 0x6B, 0xDF, 0x32, 0x5F, 0x40, 0x90, 0x9C, 0xB8, 0x3D, 0x86, 0x13, 0x26, 0xB7, 0x63, 0xF7, 0x74, 0xE8, 0x53, 0xED, 0x58, 0x20, 0x4F, 0xD9, 0x99, 0x26, 0x21, 0x37, 0xDE, 0x35, 0x76, 0xC8, 0xBC, 0xD0, 0x6E]
    res=try_inp(inp)
    res=[int(x, 16) for x in res.split()]
    print(res)
    for i in range(45):
        newinp=inp
        for char in printable:
            newinp=inp+char
            res=try_inp(newinp)
            res=[int(x, 16) for x in res.split()]
            if res[i+6] ==cipher[i+6]:
                inp=newinp
                print(inp)
                break
#hgame{D3n1ably-c0mmunicate-by-d0ing-m@g1cal-no-op!}
```

---

赛后看wp补一下这道题原理

该函数通过ptrace（详⻅https://www.man7.org/linux/man-pages/man2/ptrace.2.html）拦截⼦进 程中的每个syscall，检查该syscall的静态前驱指令是否为满⾜ [0x40-0x43] 0x0F 0x1F 0x440x7F 。若是，则读取指令并进⾏某种解码，直到遇到 [0x40-0x43] 0x0F 0x1F 0x44 0x7E

或任何不以 [0x40-0x43] 0x0F 0x1F 0x44 为前缀的指令。 这种指令序列其实是多字节nop指令通过这种序列实现了程序独自运行逻辑不发生变化，而在拦截子程序`syscall`的调用可以改变子程序的控制流

之前分析的20轮加密实际上是`chacha20`的quarter，赛时只有最后对`game`的秘钥流和flag的处理没有分析清楚

![[433164b4f3d722925edc4b2686a02fda_MD5.png]]

![[b90941f685b8d1fc498104b8646948c3_MD5.png]]

分析`RAND_POOL`可以发现实际上就是密文，我们直接交叉引用可以发现最后发生`cmp_res`的地方

![[9721e705169842131232eeb9577719c3_MD5.png]]

发现这里利用`nop_call`发送`launcher`进程执行`cmp_res`函数，并且在前面获取了`keystram`还进行了链式xor，注意还获取了一个iv

最终的exp

```
stream=bytes.fromhex("""4a 69 5b 2a f3 87 56 46 f3 07 74 c6 65 73 d6 16
45 fe d9 98 03 76 b3 6d 50 e0 96 f7 4c bc b0 a4
ea f2 dc 93 d8 38 08 a7 23 de 6b 3b 87 84 6e d1
04 4d c3 2a 56 3f ee 08 a3 d8 76 e6 6b bc 48 ca""")
ct = bytes.fromhex("""64 6A 50 17 81 7D 6F 1A 87 B1 A4 00 09 03 F8 8D F8 6B
DF 32 5F 40 90 9C B8 3D 86 13 26 B7 63 F7 74 E8 53 ED 58 20 4F D9 99 26 21
37 DE 35 76 C8 BC D0 6E""")
ct = bytes([0x46]) + ct
b = bytes([ct[i - 1] ^ ct[i] for i in range(1, len(ct))])
print(bytes([i ^ j for i, j in zip(b, stream)]))
```

### middleman|复现

![[cc2cea7cd15519f8f8f3ad9e53bccec0_MD5.png]]

加载了seccomp 禁用了除getpid外的系统调用，反调

[https://arm64.syscall.sh/](https://arm64.syscall.sh/)

![[41c6b93a57d60dcb8cbab2c42bb6ebe0_MD5.png]]

不会Android 告辞

---

接收输入并调用getpid 还有奇怪参数

![[de104710496218cefbb66d78d278a5ec_MD5.png]]

可以在`initarray`发现

![[a6207a80aca1a31019926af15b19ac41_MD5.png]]

注册了seccomp系统调用,使用`seccomp-tools`来dump cbpf字节码

```
(base) kali@kali:~/Desktop$ seccomp-tools disasm dump.bpf 
 line  CODE  JT   JF      K
=================================
 0000: 0x20 0x00 0x00 0x00000004  A = arch
 0001: 0x15 0x00 0x26 0xc00000b7  if (A != ARCH_AARCH64) goto 0040
 0002: 0x20 0x00 0x00 0x00000020  A = args[2] ;u
 0003: 0x02 0x00 0x00 0x00000000  mem[0] = A	;mem[0]=u
 0004: 0x20 0x00 0x00 0x00000028  A = args[3] ;A=v
 0005: 0x02 0x00 0x00 0x00000001  mem[1] = A	;mem[1]=v
 0006: 0x64 0x00 0x00 0x00000004  A <<= 4		;A=v<<4
 0007: 0x04 0x00 0x00 0x65766573  A += 0x65766573	;A=(v<<4)+0x65766573
 0008: 0x02 0x00 0x00 0x00000002  mem[2] = A	;mem[2]=(v<<4)+0x65766573
 0009: 0x60 0x00 0x00 0x00000001  A = mem[1]	;A=v
 0010: 0x07 0x00 0x00 0x00000000  X = A			;X=v
 0011: 0x00 0x00 0x00 0x22122122  A = 571613474 ;A=571613474
 0012: 0x0c 0x00 0x00 0x00000000  A += X		;A=v+571613474
 0013: 0x07 0x00 0x00 0x00000000  X = A			;X=v+571613474
 0014: 0x60 0x00 0x00 0x00000002  A = mem[2]	;A=(v<<4)+0x65766573
 0015: 0xac 0x00 0x00 0x00000000  A ^= X		;A=((v<<4)+0x65766573)^(v+571613474)
 0016: 0x07 0x00 0x00 0x00000000  X = A			;X=((v<<4)+0x65766573)^(v+571613474)
 0017: 0x60 0x00 0x00 0x00000000  A = mem[0]	;A=u
 0018: 0x0c 0x00 0x00 0x00000000  A += X		;A=u+((v<<4)+0x65766573)^(v+571613474)
 0019: 0x15 0x00 0x14 0x93cd6340  if (A != 2479711040) goto 0040;constraint1
 0020: 0x02 0x00 0x00 0x00000000  mem[0] = A	;mem[0]=u+((v<<4)+0x65766573)^(v+571613474)
 0021: 0x74 0x00 0x00 0x00000005  A >>= 5		;A=(u+((v<<4)+0x65766573)^(v+571613474))>>5
 0022: 0x04 0x00 0x00 0x6e6e6e6e  A += 0x6e6e6e6e ;A=((u+((v<<4)+0x65766573)^(v+571613474))>>5)+0x6e6e6e6e
 0023: 0x02 0x00 0x00 0x00000002  mem[2] = A	;mem[2]=((u+((v<<4)+0x65766573)^(v+571613474))>>5)+0x6e6e6e6e
 0024: 0x60 0x00 0x00 0x00000000  A = mem[0]	;A=u+((v<<4)+0x65766573)^(v+571613474)
 0025: 0x07 0x00 0x00 0x00000000  X = A			;X=u+((v<<4)+0x65766573)^(v+571613474)
 0026: 0x00 0x00 0x00 0x22122122  A = 571613474	;A=571613474
 0027: 0x0c 0x00 0x00 0x00000000  A += X		;A=(u+((v<<4)+0x65766573)^(v+571613474))+571613474
 0028: 0x07 0x00 0x00 0x00000000  X = A			;X=(u+((v<<4)+0x65766573)^(v+571613474))+571613474
 0029: 0x60 0x00 0x00 0x00000002  A = mem[2]	;A=((u+((v<<4)+0x65766573)^(v+571613474))>>5)+0x6e6e6e6e
 0030: 0xac 0x00 0x00 0x00000000  A ^= X		;A=(((u+((v<<4)+0x65766573)^(v+571613474))>>5)+0x6e6e6e6e)^((u+((v<<4)+0x65766573)^(v+571613474))+571613474)
 0031: 0x07 0x00 0x00 0x00000000  X = A			;X=A
 0032: 0x60 0x00 0x00 0x00000001  A = mem[1]	;A=v
 0033: 0x0c 0x00 0x00 0x00000000  A += X		;A=v+(((u+((v<<4)+0x65766573)^(v+571613474))>>5)+0x6e6e6e6e)^((u+((v<<4)+0x65766573)^(v+571613474))+571613474)
 0034: 0x15 0x00 0x05 0xb5f40d3f  if (A != 3052670271) goto 0040
 0035: 0x20 0x00 0x00 0x00000000  A = sys_number
 0036: 0x15 0x00 0x03 0x000000ac  if (A != aarch64.getpid) goto 0040
 0037: 0x20 0x00 0x00 0x00000030  A = args[4]
 0038: 0x15 0x00 0x01 0x00221221  if (A != 0x221221) goto 0040
 0039: 0x06 0x00 0x00 0x00030000  return TRAP
 0040: 0x06 0x00 0x00 0x7fff0000  return ALLOW
```

使用z3求解args[2]和args[3]

```
from z3 import *
s = Solver()
u = BitVec("u", 32)
v = BitVec("v", 32)
A = u + (((v << 4) + 0x65766573) ^ (v + 571613474))
B = v + ((LShR(A, 5) + 0x6e6e6e6e) ^ (A + 571613474))
s.add(A == 2479711040)
s.add(B == 3052670271)
assert s.check(), "No solution"
m = s.model()
u, v = int(str(m[u])), int(str(m[v]))
print(u.to_bytes(4, "little").hex(), v.to_bytes(4, "little").hex())
```

![[94ebbea4f6a2403a1b87485db25e1406_MD5.png]]

[https://bbs.kanxue.com/thread-277544.htm](https://bbs.kanxue.com/thread-277544.htm)

结合这篇博客我们可以知道`a3`实则是`ucontext`结构体的指针,使用ida恢复一下符号

之后就是使用xor计算了key，然后进行一个标准AES_ECB128

```
from pwn import xor
print(bytes.fromhex("8cd8194d 55af20ef"))
key = xor(b"Sevenlikeseccmop", bytes.fromhex("8cd8194d55af20ef"))
print(key.hex())
```

![[a0ee63a5434bcb49cee0b1353652d0cc_MD5.png]]

### FastAndFrusting|复现

[[HGAME 2025] .Net AoT分析 - Fast and frustrating-CSDN博客](https://blog.csdn.net/2401_84298456/article/details/145728937?spm=1001.2101.3001.6650.3&utm_medium=distribute.pc_relevant.none-task-blog-2%7Edefault%7EBlogOpenSearchComplete%7ECtr-3-145728937-blog-145713847.235%5Ev43%5Epc_blog_bottom_relevance_base4&depth_1-utm_source=distribute.pc_relevant.none-task-blog-2%7Edefault%7EBlogOpenSearchComplete%7ECtr-3-145728937-blog-145713847.235%5Ev43%5Epc_blog_bottom_relevance_base4&utm_relevant_index=4)

我觉得这道题目有点烂，因为即使进行patch 仍然无法进行加载正确的资源(而能找到正确的资源也需要很强大的眼力和运气),基本上就是无法动调，当然可以找到正确资源后，将正确资源patch进去，但我当时完全找不到资源就放弃了

![[7fa45939e0c5501c4b5a4020bd1a4a80_MD5.png]]

这道题目有符号表，因此完全可以静态分析

---

先正向过一遍流程

1. 检测TwoLetterISOLanguageName

![[ed4e04343b76d1964ae0487e8f9912be_MD5.png]]

2. 从控制台中读取`key`

![[f01c1ba94fe1b0a0fe09de1ae4f8bf53_MD5.png]]

3. 加载资源

![[b055a1c5a93abc833cc69e229802376b_MD5.png]]

4. 将获取的资源进行base64解码

![[7067f02fa36155b3ca08e0714d0204d9_MD5.png]]

5. 解压gzip

![[1befc2fe7a39e428ffe8d143046d726f_MD5.png]]

6. 读取json

![[c6edf7c151587fb83595eac536714166_MD5.png]]

7. 读取的线性方程组要满足的关系

![[d583f2fa89398f48c27e704101cac706_MD5.png]]

8. 调用AES解密

![[7811860bd7883f2fcd61d270e580d5cb_MD5.png]]

![[d96cd0ce10aacfb68b3065d6f32bcc10_MD5.png]]

使用CBC模式，并且使用了HKDF来进行派生KEY,再从资源中获取密文，进行base64解码

解密后输出flag

---

获取先前的资源后，我们就一步一步进行解密

先前的`base64`串实际上是`gzip`文件流的base64加密，通过特征头可以确定

```
data="H4sIABh9j2cC/21Wy47bMAz8lWDPESBS7/7KYrHYFj32VvRS9N+rGVKynQSIZUtiOCI5JPX37dfX78+vt2+393e53+L6vfzW+y3YJMzPjNl8S7UdFQxjLuAjtfstze84xep8ZOjH/fZu6mSpk/WD3mSv9QAm2WsiSJ96AAApLSZtkHOikCjc1+E44WwDDmzYuuAXXLVPnJeQ2SZlTsbAPkzKDVtUlAmaeKYOLLMgcWzblGxQjl0NKHR/+3+KSwBUcIo6V9p8woBkgdiYOntyo56Dsl1oFkJz3i6VwwGwD6vwUqNyDBLntBCZ8Tk5P15CcXZm3MFJZo7GFQycY7hGWNfGiplUkGRQa9bDnBXqJ9CQz2ELyWaFqraBmOZpSCcbsdGxiSUSI7Qn2oUL6YDi7oF6UtytsmgTAx4FApUXEiA2hAx/Mdo4N06GHQHRK6urJY8xRgxJmELwZTOhwtMXj9rIllczjaYI5ObSi4BdkTbjD5udhErn+g4mlUbDPJ6k8OSdI3BTOwy7pI+R3QLj/kvbvzCnrN0AMiKZpkaklH0y3TTTA4xhHVcXBg+++nQteQnIlnmWbdUjppZ/oajHLDHnIIVDKNOtT3wnvV6YWOzwbsqiIT4HIeAgz+PubFzlKx/pzXcafqI55GfO64Ekm+V1b4jZ2BxEnTescyUvphIDgyS4SR+idUnZXXijhcRxrGaQ/yRP27Y2hjnGRXvL502R2l/UpbBT7UgzvWaberkjUrGakVepqNnlzbMoI2ecXR8e4qS2FA3FaJdOpNF8MJLAdGGb8ngqsUCYHF877/zbTcQqETUOq+ewojRPJiYVcx0k7eReBwmrN44X0Xn044kh6+2JRmLs5eSzXCyMkvoqEPHSUeUhk7xtGbvUyZ0sVTzx0PNgGHuRwSYvT9L0XGZfNqhryT2KhxrP1PMyZA55JT177jxMa/Abj6z1KOc7FHoimytf1encNaIVBuMhN1bhplhxeVb22aPG6/vKxm47hYLnUDquKuLFATi8MEhmQg8rcWtUo7gebAiXJrWC7qS+3MTUK0PeEV0XsLyuLZVRZCFCm2+PpMsnfXm1B69jkha14Tiwm4codjOadKOV1do5F4a3Jxn90XXngHkxlU3GejR4Me91vz34bW9YeapgDFBZlFo87l7nVvfMdZt1N0s9G9O+AACoJ2+BQjQQDjxkwFr7mFBvf37++PyOi3KSRs5LjZ3ZLmCqjBjtEqJUnWJjyEdkk5hTuEaLcFpSZ7Q6rypaxyBrW4H0ZKA5NqZiTh9260VMUypoANKyuV0KylcYIxZe6nhTLaOwIaiS9IO1u0Jw/n167t9/pH9jSPgLAAA="
import base64
data = base64.b64decode(data)
import gzip
data = gzip.decompress(data).decode()
import json
data = json.loads(data)
print(data)
```

JSON数据包含一个矩阵`mat_a`和一个向量`vec_b`，结合对闭包函数的分析，这是一个线性方程组 要求满足 `mat_a`*`vec_x`=`vec_b`

```
import numpy as np
x = np.linalg.inv(data["mat_a"]) @ data["vec_b"]
print(bytes(list(map(round, x))))
#b'CompressedEmbeddedResources'
```

这就是我们输入的`key`

后面的HKDF派生出`key`和`iv`

```
using System;
using System.Security.Cryptography;
 
public class HKDFExample
{
    public static void Main()
    {
        byte[] inputKeyMaterial = System.Text.Encoding.UTF8.GetBytes("CompressedEmbeddedResources");
        byte[] salt = new byte[0];
        byte[] info = System.Text.Encoding.UTF8.GetBytes("HGAME2025");
        int outputKeyLength = 48;
        byte[] derivedKey = HKDF.DeriveKey(
            HashAlgorithmName.SHA256,
            inputKeyMaterial,
            outputKeyLength,
            salt,
            info
        );
        Console.WriteLine("派生的密钥: " + BitConverter.ToString(derivedKey).Replace("-", "").ToLower());
    }
}
```

`HKDF后做了一个key和iv的分割（32bytes + 16bytes）`

注意.NET的AES默认算法为AES-256

在资源文件找到密文![[a343643045dfec2b758a38ff48ce7e69_MD5.png]]

进行解码

![[b31fd12afb9d4fb35599078b5d78b6ff_MD5.png]]

### 神秘信号

最简单的做法就是直接把apk包内的hello常量改为h1g1a1m1e1即可

关键逻辑：

![[30799441a946322605f6b19069702a4e_MD5.png]]

当请求h1g1a1m1e1文件就会走入解密流程

![[d5c28705c9100a9a92f26e4d27a7e4e5_MD5.jpg]]