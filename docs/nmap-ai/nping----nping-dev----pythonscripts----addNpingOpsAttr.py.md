# `nmap\nping\nping-dev\pythonscripts\addNpingOpsAttr.py`

```
# 从用户输入中获取方法名
methname = raw_input("Method name: ")
# 从用户输入中获取属性名
attrname = raw_input("Attr name: ")
# 从用户输入中获取属性类型
attrtype = raw_input("Attr type: ")

# 打开文件 "Output.txt"，以追加模式写入
o = open("Output.txt","a") 
# 遍历文件 "TemplateNpingOps.txt" 的每一行
for line in open("TemplateNpingOps.txt"):
   # 替换每行中的 "ATTRNAME" 为用户输入的属性名
   line = line.replace("ATTRNAME",attrname)
   # 替换每行中的 "METHNAME" 为用户输入的方法名
   line = line.replace("METHNAME",methname)
   # 替换每行中的 "TYPE" 为用户输入的属性类型
   line = line.replace("TYPE",attrtype)
   # 将替换后的行写入到文件 "Output.txt"
   o.write(line) 
# 关闭文件 "Output.txt"
o.close()
```