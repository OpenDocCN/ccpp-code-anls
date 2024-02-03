# `nmap\zenmap\install_scripts\macosx\zenmap_launcher.py`

```cpp
# 导入所需的模块
from os.path import join, dirname, abspath, normpath
import sys, os
import platform

# 获取程序所在路径
bundlepath = sys.argv[0]

# 设置 bundle_contents 为程序所在路径下的 Contents 文件夹路径
bundle_contents = join(bundlepath, 'Contents')
# 设置 bundle_res 为 bundle_contents 下的 Resources 文件夹路径
bundle_res = join(bundle_contents, 'Resources')

# 设置其他路径
bundle_lib = join(bundle_res, 'lib')
bundle_bin = join(bundle_res, 'bin')
bundle_data = join(bundle_res, 'share')
bundle_etc = join(bundle_res, 'etc')

# 设置环境变量
os.environ['XDG_DATA_DIRS'] = bundle_data
os.environ['DYLD_LIBRARY_PATH'] = bundle_lib
os.environ['LD_LIBRARY_PATH'] = bundle_lib
os.environ['GTK_DATA_PREFIX'] = bundle_res
os.environ['GTK_EXE_PREFIX'] = bundle_res
os.environ['GTK_PATH'] = bundle_res

os.environ['PANGO_RC_FILE'] = join(bundle_etc, 'pango', 'pangorc')
os.environ['PANGO_SYSCONFDIR'] = bundle_etc
os.environ['PANGO_LIBDIR'] = bundle_lib
os.environ['GDK_PIXBUF_MODULE_FILE'] = join(bundle_lib, 'gdk-pixbuf-2.0', '2.10.0', 'loaders.cache')

# 根据平台版本设置环境变量
if int(platform.release().split('.')[0]) > 10:
    os.environ['GTK_IM_MODULE_FILE'] = join(bundle_etc, 'gtk-3.0', 'gtk.immodules')

os.environ['GI_TYPELIB_PATH'] = join(bundle_lib, 'girepository-1.0')

# 设置 $PYTHON 指向 bundle_res
PYVER = 'python3.10'
sys.path.append(bundle_res)

# 设置环境变量
os.environ['USERPROFILE'] = os.environ['HOME']
os.environ['APPDATA'] = join(os.environ['HOME'], 'Library', 'Application Support')

# 导入并运行主函数所在的模块
from zenmapGUI import App
App.run()
```