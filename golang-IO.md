
## symlink
windows:
快捷方式的os.FileInfo， IsDir() false, Name()多了后缀.lnk
ioutil.ReadDir()不会读取目录快捷方式

linux:


## http
the web server invokes each handler in a new goroutine, so handlers must take precautions such as locking when accessing variables that other goroutines, including other requests to the same handler, may be accessing.
