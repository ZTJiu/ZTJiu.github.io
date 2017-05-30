---
title: "Linux下文件监控"
---

# Linux下监控文件系统

Linux的后台程序通常在机器没有问题的情况下，需要长期运行（比如说数个月，甚至是数年）。但是，程序的配置文件有时候是需要定期作调整。为了不影响程序对外服务（不重启），动态加载配置文件是一种非常常见的需求。通过监控某个文件的创建、删除和修改等事件，可以很方便做出对应的动作（比如说reload）。

## 1. Linux下监控文件系统的常用方法

监控配置文件或配置文件目录的变化，一种可行的方法是程序启动的时候记录下文件（或目录）的修改时间，周期性检查（比如说一秒一次）文件是否已经被修改，来决定是否需要重新加载配置文件。

另一种更为优雅的办法是使用Linux系统从内核层面支持的系统API dnotify、inotify或者fanotify。inotify API提供一个文件描述符，可以在该文件描述符上注册对指定的文件或者目录的文件系统事件（文件删除、文件修改和文件创建），然后通过read系统调用读取该文件描述法上的事件。

## 2. 使用stat或fstat监控Linux文件系统

通过周期性地获取被监控文件的状态，stat和fstat可以帮助用户监控指定文件的状态。

```c
	int stat(const char *path, struct stat *buf);
    int fstat(int fd, struct stat *buf);
	struct stat {
           dev_t     st_dev;     /* ID of device containing file */
           ino_t     st_ino;     /* inode number */
           mode_t    st_mode;    /* protection */
           nlink_t   st_nlink;   /* number of hard links */
           uid_t     st_uid;     /* user ID of owner */
           gid_t     st_gid;     /* group ID of owner */
           dev_t     st_rdev;    /* device ID (if special file) */
           off_t     st_size;    /* total size, in bytes */
           blksize_t st_blksize; /* blocksize for file system I/O */
           blkcnt_t  st_blocks;  /* number of 512B blocks allocated */
           time_t    st_atime;   /* time of last access */
           time_t    st_mtime;   /* time of last modification */
           time_t    st_ctime;   /* time of last status change */
   };
```
文件状态结构体struct stat的st_mtime字段记录了path对应的文件的最后修改时间，只用周期性地检查st_mtime的值，就可以监控文件是否被修改了。

开启一个单独的线程，并在线程中注册一个回调函数，当文件改变了以后，就调用注册的回调函数重新加载监控的目标文件。一个简单的例子如下：

```c++
#include <iostream>
#include <string>
#include <thread>
#include <functional>
#include <ev.h>

#include <sys/stat.h>
#include <sys/types.h>
#include <unistd.h>
#include <string.h>
#include <errno.h>

typedef std::function<void (std::string &)> Notifier;
class ConfWatcher {
public:
    explicit ConfWatcher(const std::string &conf_path, Notifier notifier)
    : conf_path_(conf_path), notifier_(notifier), wait_sec_(1), stop_(false)
    {  }

    bool Init() {
        if (access(conf_path_.c_str(), F_OK) != 0) {
            std::cout << "conf file " << conf_path_ << " doesn't exist" << std::endl;
            return false;
        }
        if (!update_last_mod_time()) {
            return false;
        }
        thread_ = std::thread([this] () {
            std::cout << "starting..." << std::endl;
            time_t now = 0;
            while (!stop_) {
                now = time(nullptr);
                time_t delay = now - last_detect_time_;
                if (delay < wait_sec_) {
                    //std::cout << "sleep " << wait_sec_ - delay << " seconds" << std::endl;
                    sleep(wait_sec_ - delay);
                }
                time_t mod_time = last_mod_time_;
                update_last_mod_time();
                if (mod_time != last_mod_time_) {
                    std::cout << "target file " << conf_path_ <<
                    " has been modified, so reload it" << std::endl;
                    notifier_(conf_path_);
                }
            }
            std::cout << "stopping..." << std::endl;
        });
        return true;
    }
    bool Stop() { stop_ = true; }
    bool Wait() { thread_.join(); }

private:
    bool update_last_mod_time() {
        struct stat f_stat = {0};
        if (stat(conf_path_.c_str(), &f_stat) == -1) {
            std::cout << "update_last_mod_time() failed on conf_file "
                << conf_path_ << " ,error: " << strerror(errno) << std::endl;
            return false;
        }
        last_mod_time_ = f_stat.st_mtime;
        last_detect_time_ = time(nullptr);
        return true;
    }

private:
    std::string conf_path_;
    int wait_sec_;
    bool stop_;
    time_t last_mod_time_;
    time_t last_detect_time_;
    std::thread thread_;
    Notifier notifier_;
};

int main(int argc, char **argv) {
    if (argc != 2) {
        std::cout << "Usage: " << argv[0] << " target-file-path" << std::endl;
        return -1;
    }
    auto notifier = [] (const std::string &conf_file_path) {
        std::cout << "watched file " << conf_file_path
            << " has been modified, maybe you need reload it" << std::endl;
    };
    ConfWatcher watcher(argv[1], notifier);
    if (!watcher.Init()) {
        return -1;
    }
    watcher.Wait();
    return 0;
}
```
编译：g++ -std=c++11  -lpthread main.cpp 



## 3. dnotify监控Linux文件系统

dnofigy是Linux kernel 2.4.0开始支持的一个系统API。它提供了非常有限的方式来和内核交互，以便获取指定的目录下文件的修改事件。dnotify的文件监控室通过fcntl的F_NOTIFY选项来实现的。因此它需要使用一个用open系统调用返回的一个文件描述符。可监控的事件有：文件读取、文件创建、文件内容修改、文件属性修改和文件重命名。dnotify的事件通常是通过SIGIO信号来得到通知的。

dnotify有以下缺陷，因此使用起来不是很方便：无法监控单个文件；被监控文件夹所在的文件系统分区在进程退出前无法卸载；文件事件发生后，需要调用stat系统调用来检查到底是那个文件被修改了。

## 4. 使用inotify监控Linux文件系统

inotify是Linux 2.6.13引入的一个inode监控系统。它的API提供了监控单个文件或者目录的机制，当监控目录时，它可以返回目录本身的事件和目录下文件的事件。因此inotify可以完全替代dnotify。

inotigy可以监控以下事件（在此只列出部分，更详细信息可以man 7 inotify查看）：

```c
IN_ACCESS         File was accessed (read) (*). 
IN_ATTRIB         Metadata changed, e.g., permissions, timestamps, extended attributes, link count (since Linux 2.6.25), UID, GID, etc. (*).
IN_CLOSE_WRITE    File opened for writing was closed (*).
IN_CLOSE_NOWRITE  File not opened for writing was closed (*).
IN_CREATE         File/directory created in watched directory (*).
IN_DELETE         File/directory deleted from watched directory (*).
IN_DELETE_SELF    Watched file/directory was itself deleted.
IN_MODIFY         File was modified (*).
IN_MOVE_SELF      Watched file/directory was itself moved.
IN_MOVED_FROM     Generated for the directory containing the old filename when a file is renamed (*).
IN_MOVED_TO       Generated for the directory containing the new filename when a file is renamed (*).
IN_OPEN           File was opened (*).
```

inotify的接口inotify_init()返回一个文件描述符，然后调用inotify_add_watch()来添加监控的目录或文件及其对应的感兴趣的事件。对于不感兴趣的文件或目录，可以使用inotify_rm_watch()来取消关注。注册了文件监控后，可以使用select、poll或者epoll来检测inotify_init()返回的文件描述符是否有可读事件，然后使用read系统调用将可读事件读取到struct inotify_event里。

```c
struct inotify_event {
    int      wd;       /* Watch descriptor */
    uint32_t mask;     /* Mask of events */
    uint32_t cookie;   /* Unique cookie associating related events (for rename(2)) */
    uint32_t len;      /* Size of name field */
    char     name[];   /* Optional null-terminated name */
};
```
正常情况下，read可以读取到若干个struct inotify_event，然后逐个遍历就能知道是哪个文件（或者目录）上发生了什么事件。下面是一个简单的使用方法：

```c++
//#pragma once
#include <stdlib.h>
#include <errno.h>
#include <string.h>
#include <sys/types.h>
#include <unistd.h>
#include <sys/inotify.h>
#include <string>
#include <unordered_map>
#include <vector>
#include <iostream>
#include <thread>
#include <ev.h>

struct fs_io {
    struct ev_io io;
    void * data;
};

class FsMonitor {
public:
    ~FsMonitor() {
        if (fd_ != -1) {
            close(fd_);
        }
    }

    // register file/directory path and event mask.
    bool Init(const std::vector<std::pair<std::string,uint32_t>> &item_list);
    bool Run();
    bool Stop();

private:
    static void HandleEvents(struct ev_loop *loop, struct ev_io *watcher, int revents);
    bool RemoveItem(const std::string &item_name);
    bool ModifyItem(const std::string &item_name, uint32_t mask);

private:
    std::unordered_map<std::string,int> item_wd_;
    int fd_;

    struct ev_loop *loop_;
    struct fs_io watcher_;
    std::thread thread_;
};


#define EVENT_SIZE  (sizeof (struct inotify_event))
#define BUF_LEN     (1024 * (EVENT_SIZE + 16))

bool FsMonitor::Init(const std::vector<std::pair<std::string,uint32_t>> &item_list)
{
    fd_ = inotify_init1(IN_NONBLOCK);
    if (fd_ < 0) {
        perror("inotify_init1");
        return false;
    }

    for (auto item : item_list) {
        int wd = inotify_add_watch(fd_, item.first.c_str(), item.second);
        if (wd == -1) {
            std::cout << "inotify_add_watch() item " << item.first << " failed:" << strerror(errno)  << std::endl;
            return false;
        }
        item_wd_.insert(std::pair<std::string,int>(item.first, wd));
    }

	loop_ = EV_DEFAULT;

    watcher_.data = this;
	ev_io_init (&watcher_.io, HandleEvents, fd_, EV_READ);
	ev_io_start (loop_, &watcher_.io);

    return true;
}

bool FsMonitor::Run()
{
    thread_ = std::thread([this] () {
        ev_run(loop_, 0);
    });
    return true;
}

bool FsMonitor::Stop()
{
    ev_break(loop_, EVBREAK_ALL);
    return true;
}

void FsMonitor::HandleEvents(struct ev_loop *loop, struct ev_io *watcher, int revents)
{
    struct fs_io * fs_watcher = (struct fs_io *)watcher;
    FsMonitor * fs_monitor = (FsMonitor *)fs_watcher->data;

    int length, i = 0;
    char buffer[BUF_LEN];
    length = read(fs_monitor->fd_, buffer, BUF_LEN);
    if (length < 0) {
        perror("read");
        return;
    }

    while (i < length) {
        struct inotify_event *event = (struct inotify_event *) &buffer[i];
        if (event->len) {
            if (event->mask & IN_CREATE) {
                if (event->mask & IN_ISDIR) {
                    std::cout <<"The directory " << event->name <<" was created." << std::endl;
                }
                else {
                    std::cout <<"The file " << event->name << " was created." << std::endl;
                }
            }
            else if (event->mask & IN_DELETE) {
                if (event->mask & IN_ISDIR) {
                    std::cout <<"The directory " << event->name << " was deleted." << std::endl;
                }
                else {
                    std::cout <<"The file " << event->name << " was deleted." << std::endl;
                }
            }
            else if (event->mask & IN_MODIFY) {
                if (event->mask & IN_ISDIR) {
                    std::cout <<"The directory " << event->name << " was modified." << std::endl;
                }
                else {
                    std::cout <<"The file " << event->name << " was modified." << std::endl;
                }
            }
        }
        i += EVENT_SIZE + event->len;
    }
}

bool FsMonitor::RemoveItem(const std::string &item_name) {
    auto find_item = item_wd_.find(item_name);
    if (find_item == item_wd_.end()) {
        std::cout << "no item " << item_name << " to remove" << std::endl;
        return false;
    }
    int wd = find_item->second;
    inotify_rm_watch(fd_, wd);
    return true;
}

bool FsMonitor::ModifyItem(const std::string &item_name, uint32_t mask) {
    // remove and add
    return true;
}

int main(int argc, char **argv) {
    FsMonitor fs_monitor;
    std::vector<std::pair<std::string,uint32_t>> item_list;
    item_list.emplace_back(std::pair<std::string,uint32_t>("/tmp", IN_MODIFY | IN_CREATE | IN_DELETE));
    if (fs_monitor.Init(item_list) == false) {
        return -1;
    }
    std::cout << "Starting..." << std::endl;
    fs_monitor.Run();
    while (true) {
        usleep(1000 * 1000);
    }
    std::cout << "End..." << std::endl;

    return 0;
}

```

编译：g++ -g -std=c++11 inotify.cpp -lev -pthread



## 5. fanotify 

fanotify需要root权限才能使用，应用场景比较少，这里就不介绍啦。有兴趣的可以查看给出的参考文章。



References:

1. http://www.lanedo.com/filesystem-monitoring-linux-kernel/
2. http://www.ibm.com/developerworks/library/l-ubuntu-inotify/index.html

