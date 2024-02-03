# `nmap\zenmap\install_scripts\utils\get_deps.py`

```cpp
#!/usr/bin/env python3

# 导入模块
import modulefinder
import configparser
import sys
import os.path
import site
import encodings

# 需要检查的站点包依赖
site_package_deps = ("gi", "cairo")

# 不需要的模块
pyd_remove = ("_decimal", "_ssl", "_testcapi", "_hashlib")

# 获取模块路径
def module_paths(mods):
    for m in mods:
        # 如果模块在不需要的模块列表中，则跳过
        if m.__name__ in pyd_remove:
            continue
        # 如果模块有文件路径，并且路径以系统前缀开头，则返回模块文件路径
        elif getattr(m, "__file__", None) and m.__file__.startswith(sys.prefix):
            yield m.__file__

# 获取依赖
def get_deps():
    # 获取站点包路径
    sitedirs = site.getsitepackages()
    # 将站点包依赖的文件添加到集合中
    files = set(os.path.join(sitedirs[0], name) for name in site_package_deps)

    # 这些项目被 modulefinder 忽略
    files.add(encodings.__path__[0])  # 所有编码，以防万一
    # 获取站点包和内置站点包的模块路径
    for path in module_paths((site, site._sitebuiltins)):
        files.add(path)

    # 使用 modulefinder 获取其余依赖
    mfind = modulefinder.ModuleFinder()
    # 运行脚本并获取模块路径
    mfind.run_script(os.path.normpath(__file__ + '/../../../zenmapGUI/App.py'))
    for path in module_paths(mfind.modules.values()):
        parent = os.path.dirname(path)
        found_parent = False
        # 如果父目录已经包含在集合中，则不添加文件路径
        while parent not in sys.path and len(parent) > 2:
            if parent in files:
                found_parent = True
                break
            parent = os.path.dirname(parent)
        if not found_parent:
            files.add(path)
    return files

# 读取配置文件
def read_cfg(filename):
    cfg = configparser.ConfigParser()
    cfg.read(filename)
    return cfg

# 写入配置文件
def write_cfg(cfg, filename):
    with open(filename, "w") as f:
        cfg.write(f)

# 更新配置文件
def update_cfg(cfg, files):
    # 将文件路径添加到配置文件中
    filestr = "\nmingw*".join((f.removeprefix(sys.prefix) for f in files))
    oldvalue = cfg.get('bundle', 'nodelete')
    cfg.set('bundle', 'nodelete', oldvalue + "\nmingw*" + filestr)

# 写入 XML 文件
def write_xml(filename, files):
    # 以写入模式打开文件
    with open(filename, "w") as f:
        # 遍历文件列表
        for file in files:
            # 构建文件名，去掉文件名前缀
            fname = r"${prefix}" + file.removeprefix(sys.prefix)
            # 设置默认格式为<data>...</data>
            fmt = "<data>{}</data>"
            # 如果文件以.so结尾，则格式为<binary>...</binary>
            if file.endswith(".so"):
                fmt = "<binary>{}</binary>"
            # 将格式化后的文件名写入文件
            print(fmt.format(fname), file=f)
# 如果当前模块被直接执行
if __name__ == "__main__":
    # 获取依赖文件列表
    files = get_deps()
    # 如果操作系统是 Windows
    if sys.platform == "win32":
        # 读取配置文件
        cfg = read_cfg(sys.argv[2])
        # 更新配置文件
        update_cfg(cfg, files)
        # 写入配置文件
        write_cfg(cfg, sys.argv[1])
    # 如果操作系统是 macOS
    elif sys.platform == "darwin":
        # 写入 XML 文件
        write_xml(sys.argv[1], files)
    # 如果是其它操作系统
    else:
        # 抛出未实现的错误
        raise NotImplementedError
```