# `nmap\libz\contrib\dotzlib\DotZLib\UnitTests.cs`

```cpp
//
// © Copyright Henrik Ravn 2004
// 版权声明，版权归属于 Henrik Ravn 2004
//
// Use, modification and distribution are subject to the Boost Software License, Version 1.0.
// (See accompanying file LICENSE_1_0.txt or copy at http://www.boost.org/LICENSE_1_0.txt)
// 使用、修改和分发受 Boost 软件许可证 1.0 版的约束
// （请参阅随附的文件 LICENSE_1_0.txt 或在 http://www.boost.org/LICENSE_1_0.txt 复制）
//

using System;
using System.Collections;
using System.IO;

// uncomment the define below to include unit tests
//#define nunit
#if nunit
using NUnit.Framework;

// Unit tests for the DotZLib class library
// ----------------------------------------
// DotZLib 类库的单元测试
//
// Use this with NUnit 2 from http://www.nunit.org
// 使用 NUnit 2 进行测试，网址：http://www.nunit.org
//

namespace DotZLibTests
{
    using DotZLib;

    // helper methods
    // 辅助方法
    internal class Utils
    {
        // 比较两个字节数组是否相等
        public static bool byteArrEqual( byte[] lhs, byte[] rhs )
        {
            if (lhs.Length != rhs.Length)
                return false;
            for (int i = lhs.Length-1; i >= 0; --i)
                if (lhs[i] != rhs[i])
                    return false;
            return true;
        }

    }


    [TestFixture]
    // CircBufferTests 类的单元测试
    public class CircBufferTests
    {
        #region Circular buffer tests
        # 单个放入和取出测试
        [Test]
        public void SinglePutGet()
        {
            # 创建一个大小为10的循环缓冲区对象
            CircularBuffer buf = new CircularBuffer(10);
            # 断言缓冲区大小为0
            Assert.AreEqual( 0, buf.Size );
            # 断言从空缓冲区中取出数据为-1
            Assert.AreEqual( -1, buf.Get() );
    
            # 放入数据1到缓冲区中
            Assert.IsTrue(buf.Put( 1 ));
            # 断言缓冲区大小为1
            Assert.AreEqual( 1, buf.Size );
            # 断言从缓冲区中取出的数据为1
            Assert.AreEqual( 1, buf.Get() );
            # 断言缓冲区大小为0
            Assert.AreEqual( 0, buf.Size );
            # 断言再次从空缓冲区中取出数据为-1
            Assert.AreEqual( -1, buf.Get() );
        }
    
        # 块放入和取出测试
        [Test]
        public void BlockPutGet()
        {
            # 创建一个大小为10的循环缓冲区对象
            CircularBuffer buf = new CircularBuffer(10);
            # 创建一个包含1到10的数组
            byte[] arr = {1,2,3,4,5,6,7,8,9,10};
            # 断言成功放入10个数据到缓冲区中
            Assert.AreEqual( 10, buf.Put(arr,0,10) );
            # 断言缓冲区大小为10
            Assert.AreEqual( 10, buf.Size );
            # 断言无法再放入数据11到缓冲区中
            Assert.IsFalse( buf.Put(11) );
            # 断言从缓冲区中取出的数据为1
            Assert.AreEqual( 1, buf.Get() );
            # 断言成功放入数据11到缓冲区中
            Assert.IsTrue( buf.Put(11) );
    
            # 创建一个arr的副本arr2
            byte[] arr2 = (byte[])arr.Clone();
            # 断言成功从缓冲区中取出9个数据到arr2中
            Assert.AreEqual( 9, buf.Get(arr2,1,9) );
            # 断言arr和arr2相等
            Assert.IsTrue( Utils.byteArrEqual(arr,arr2) );
        }
    
        #endregion
    }
    
    [TestFixture]
    public class ChecksumTests
    {
        // CRC32 测试
        [Test]
        public void CRC32_Null()
        {
            // 创建 CRC32Checksum 对象
            CRC32Checksum crc32 = new CRC32Checksum();
            // 断言 CRC32 值为 0
            Assert.AreEqual( 0, crc32.Value );
    
            // 创建 CRC32Checksum 对象，初始值为 1
            crc32 = new CRC32Checksum(1);
            // 断言 CRC32 值为 1
            Assert.AreEqual( 1, crc32.Value );
    
            // 创建 CRC32Checksum 对象，初始值为 556
            crc32 = new CRC32Checksum(556);
            // 断言 CRC32 值为 556
            Assert.AreEqual( 556, crc32.Value );
        }
    
        [Test]
        public void CRC32_Data()
        {
            // 创建 CRC32Checksum 对象
            CRC32Checksum crc32 = new CRC32Checksum();
            // 定义字节数组
            byte[] data = { 1,2,3,4,5,6,7 };
            // 更新 CRC32 值
            crc32.Update(data);
            // 断言 CRC32 值为 0x70e46888
            Assert.AreEqual( 0x70e46888, crc32.Value  );
    
            // 创建 CRC32Checksum 对象
            crc32 = new CRC32Checksum();
            // 更新 CRC32 值
            crc32.Update("penguin");
            // 断言 CRC32 值为 0x0e5c1a120
            Assert.AreEqual( 0x0e5c1a120, crc32.Value );
    
            // 创建 CRC32Checksum 对象，初始值为 1
            crc32 = new CRC32Checksum(1);
            // 更新 CRC32 值
            crc32.Update("penguin");
            // 断言 CRC32 值为 0x43b6aa94
            Assert.AreEqual(0x43b6aa94, crc32.Value);
        }
        // CRC32 测试结束
    
        // Adler 测试
        [Test]
        public void Adler_Null()
        {
            // 创建 AdlerChecksum 对象
            AdlerChecksum adler = new AdlerChecksum();
            // 断言 Adler 值为 0
            Assert.AreEqual(0, adler.Value);
    
            // 创建 AdlerChecksum 对象，初始值为 1
            adler = new AdlerChecksum(1);
            // 断言 Adler 值为 1
            Assert.AreEqual( 1, adler.Value );
    
            // 创建 AdlerChecksum 对象，初始值为 556
            adler = new AdlerChecksum(556);
            // 断言 Adler 值为 556
            Assert.AreEqual( 556, adler.Value );
        }
    
        [Test]
        public void Adler_Data()
        {
            // 创建 AdlerChecksum 对象，初始值为 1
            AdlerChecksum adler = new AdlerChecksum(1);
            // 定义字节数组
            byte[] data = { 1,2,3,4,5,6,7 };
            // 更新 Adler 值
            adler.Update(data);
            // 断言 Adler 值为 0x5b001d
            Assert.AreEqual( 0x5b001d, adler.Value  );
    
            // 创建 AdlerChecksum 对象
            adler = new AdlerChecksum();
            // 更新 Adler 值
            adler.Update("penguin");
            // 断言 Adler 值为 0x0bcf02f6
            Assert.AreEqual(0x0bcf02f6, adler.Value );
    
            // 创建 AdlerChecksum 对象，初始值为 1
            adler = new AdlerChecksum(1);
            // 更新 Adler 值
            adler.Update("penguin");
            // 断言 Adler 值为 0x0bd602f7
            Assert.AreEqual(0x0bd602f7, adler.Value);
        }
        // Adler 测试结束
    }
    {
        #region Info tests
        # 测试 Info 类的版本信息
        [Test]
        public void Info_Version()
        {
            # 创建 Info 对象
            Info info = new Info();
            # 断言版本号为 "1.2.13"
            Assert.AreEqual("1.2.13", Info.Version);
            # 断言 SizeOfUInt 为 32
            Assert.AreEqual(32, info.SizeOfUInt);
            # 断言 SizeOfULong 为 32
            Assert.AreEqual(32, info.SizeOfULong);
            # 断言 SizeOfPointer 为 32
            Assert.AreEqual(32, info.SizeOfPointer);
            # 断言 SizeOfOffset 为 32
            Assert.AreEqual(32, info.SizeOfOffset);
        }
        #endregion
    }
    
    [TestFixture]
    public class DeflateInflateTests
    {
        #region Deflate tests
        # 压缩测试区域开始
    
        [Test]
        # 测试压缩初始化
        public void Deflate_Init()
        {
            # 使用默认压缩级别创建压缩对象
            using (Deflater def = new Deflater(CompressLevel.Default))
            {
                # 在此处执行压缩初始化的操作
            }
        }
    
        private ArrayList compressedData = new ArrayList();
        private uint adler1;
    
        private ArrayList uncompressedData = new ArrayList();
        private uint adler2;
    
        public void CDataAvail(byte[] data, int startIndex, int count)
        {
            # 将压缩后的数据添加到压缩数据列表中
            for (int i = 0; i < count; ++i)
                compressedData.Add(data[i+startIndex]);
        }
    
        [Test]
        # 测试压缩数据
        public void Deflate_Compress()
        {
            # 清空压缩数据列表
            compressedData.Clear();
    
            # 创建测试数据并填充
            byte[] testData = new byte[35000];
            for (int i = 0; i < testData.Length; ++i)
                testData[i] = 5;
    
            # 使用指定压缩级别创建压缩对象
            using (Deflater def = new Deflater((CompressLevel)5))
            {
                # 注册数据可用事件处理程序
                def.DataAvailable += new DataAvailableHandler(CDataAvail);
                # 添加测试数据到压缩对象
                def.Add(testData);
                # 结束压缩
                def.Finish();
                # 获取校验和
                adler1 = def.Checksum;
            }
        }
        #endregion
        # 压缩测试区域结束
    
        #region Inflate tests
        # 解压缩测试区域开始
    
        [Test]
        # 测试解压缩初始化
        public void Inflate_Init()
        {
            # 使用默认参数创建解压缩对象
            using (Inflater inf = new Inflater())
            {
                # 在此处执行解压缩初始化的操作
            }
        }
    
        private void DDataAvail(byte[] data, int startIndex, int count)
        {
            # 将解压缩后的数据添加到解压缩数据列表中
            for (int i = 0; i < count; ++i)
                uncompressedData.Add(data[i+startIndex]);
        }
    
        [Test]
        # 测试解压缩数据
        public void Inflate_Expand()
        {
            # 清空解压缩数据列表
            uncompressedData.Clear();
    
            # 使用默认参数创建解压缩对象
            using (Inflater inf = new Inflater())
            {
                # 注册数据可用事件处理程序
                inf.DataAvailable += new DataAvailableHandler(DDataAvail);
                # 添加压缩数据到解压缩对象
                inf.Add((byte[])compressedData.ToArray(typeof(byte)));
                # 结束解压缩
                inf.Finish();
                # 获取校验和
                adler2 = inf.Checksum;
            }
            # 断言校验和是否相等
            Assert.AreEqual( adler1, adler2 );
        }
        #endregion
        # 解压缩测试区域结束
    }
    
    [TestFixture]
    public class GZipStreamTests
    {
        #region GZipStream test
        // 测试 GZipStream 的写入和读取功能
        [Test]
        public void GZipStream_WriteRead()
        {
            // 使用 GZipStream 对象进行写入操作，使用最佳压缩级别
            using (GZipStream gzOut = new GZipStream("gzstream.gz", CompressLevel.Best))
            {
                // 创建二进制写入器
                BinaryWriter writer = new BinaryWriter(gzOut);
                // 写入字符串 "hi there"
                writer.Write("hi there");
                // 写入数学常数 PI
                writer.Write(Math.PI);
                // 写入整数 42
                writer.Write(42);
            }

            // 使用 GZipStream 对象进行读取操作
            using (GZipStream gzIn = new GZipStream("gzstream.gz"))
            {
                // 创建二进制读取器
                BinaryReader reader = new BinaryReader(gzIn);
                // 读取字符串并进行断言
                string s = reader.ReadString();
                Assert.AreEqual("hi there",s);
                // 读取双精度浮点数并进行断言
                double d = reader.ReadDouble();
                Assert.AreEqual(Math.PI, d);
                // 读取整数并进行断言
                int i = reader.ReadInt32();
                Assert.AreEqual(42,i);
            }

        }
        #endregion
    }
}
# 结束一个代码块或函数的定义
#endif
# 结束一个条件编译指令的定义
```