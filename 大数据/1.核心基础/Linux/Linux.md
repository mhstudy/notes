# Linux & Shell

> 📌 **学习重点**：大数据开发中常用的 Linux 命令和 Shell 脚本

---

## 第1章 Linux 常用命令 🔥

### 1.1 文件与目录 🔥

```bash
# 目录操作
ls -la                  # 列出所有文件（含隐藏文件）
cd /opt/module          # 切换目录
pwd                     # 显示当前路径
mkdir -p /a/b/c         # 递归创建目录
rmdir dir1              # 删除空目录

# 文件操作
touch file.txt          # 创建空文件
cp -r src/ dst/         # 递归复制
mv old.txt new.txt      # 移动/重命名
rm -rf dir/             # 强制递归删除（慎用！）

# 查看文件
cat file.txt            # 查看全部内容
more file.txt           # 分页查看
less file.txt           # 分页查看（可上下翻页）
head -n 20 file.txt     # 查看前20行
tail -f log.txt         # 实时跟踪日志（常用！）🔥
tail -n 100 log.txt     # 查看最后100行

# 查找
find / -name "*.log"              # 按名查找
find /opt -size +100M             # 查找大于100M的文件
grep -rn "error" /var/log/        # 递归搜索关键词 🔥
grep -i "pattern" file.txt        # 忽略大小写
```

### 1.2 权限管理 ⭐

```bash
chmod 755 script.sh     # rwxr-xr-x
chmod +x script.sh      # 添加执行权限
chown user:group file    # 修改所有者
```

### 1.3 系统管理 🔥

```bash
# 进程
ps -ef | grep java       # 查看 Java 进程 🔥
ps aux                    # 查看所有进程
kill -9 PID               # 强制杀进程
top                       # 实时监控资源
free -h                   # 查看内存
df -h                     # 查看磁盘

# 网络
netstat -nltp             # 查看端口监听 🔥
curl http://localhost:8080
ping hadoop102
ssh hadoop102             # 远程登录
scp -r file user@host:/path  # 远程复制

# 服务
systemctl start/stop/restart/status service_name
systemctl enable service_name   # 开机自启
```

### 1.4 管道与重定向 🔥

```bash
cat file.txt | grep "error" | wc -l       # 统计错误行数
echo "hello" > file.txt                    # 覆盖写入
echo "world" >> file.txt                   # 追加写入
command 2>&1 | tee log.txt                 # 同时输出和保存
ls /not_exist 2>/dev/null                  # 错误重定向到空
```

---

## 第2章 Shell 脚本 🔥

### 2.1 基础语法

```bash
#!/bin/bash
# 变量
name="大数据"
echo "Hello, ${name}"
echo "参数1: $1, 参数个数: $#, 所有参数: $@"

# 只读变量
readonly PI=3.14

# 命令替换
current_date=$(date +%Y-%m-%d)
file_count=`ls | wc -l`
```

### 2.2 条件判断

```bash
#!/bin/bash
# 数字比较: -eq -ne -gt -ge -lt -le
if [ $1 -gt 18 ]; then
    echo "成年"
elif [ $1 -eq 18 ]; then
    echo "刚好18"
else
    echo "未成年"
fi

# 字符串比较: = != -z(空) -n(非空)
if [ "$name" = "admin" ]; then
    echo "管理员"
fi

# 文件判断: -f(文件) -d(目录) -e(存在) -r(可读) -w(可写) -x(可执行)
if [ -f /etc/hosts ]; then
    echo "文件存在"
fi
```

### 2.3 循环

```bash
# for 循环
for i in 1 2 3 4 5; do
    echo $i
done

for i in $(seq 1 10); do
    echo $i
done

for ((i=0; i<10; i++)); do
    echo $i
done

# while 循环
count=0
while [ $count -lt 5 ]; do
    echo $count
    count=$((count + 1))
done
```

### 2.4 函数

```bash
#!/bin/bash
function add() {
    echo $(($1 + $2))
}

result=$(add 10 20)
echo "结果: $result"
```

### 2.5 大数据集群管理脚本（实战）🔥

```bash
#!/bin/bash
# 集群分发脚本 xsync
if [ $# -lt 1 ]; then
    echo "参数不够！用法: xsync <file>"
    exit
fi

for host in hadoop102 hadoop103 hadoop104; do
    echo "========== $host =========="
    for file in $@; do
        if [ -e $file ]; then
            pdir=$(cd -P $(dirname $file); pwd)
            fname=$(basename $file)
            ssh $host "mkdir -p $pdir"
            rsync -av $pdir/$fname $host:$pdir
        else
            echo "$file 不存在！"
        fi
    done
done
```

```bash
#!/bin/bash
# 集群执行命令脚本 xcall
for host in hadoop102 hadoop103 hadoop104; do
    echo "========== $host =========="
    ssh $host "$@"
done
```

```bash
#!/bin/bash
# Hadoop 集群启停脚本
case $1 in
"start")
    echo "===== 启动 Hadoop 集群 ====="
    ssh hadoop102 "/opt/module/hadoop/sbin/start-dfs.sh"
    ssh hadoop103 "/opt/module/hadoop/sbin/start-yarn.sh"
    ;;
"stop")
    echo "===== 停止 Hadoop 集群 ====="
    ssh hadoop103 "/opt/module/hadoop/sbin/stop-yarn.sh"
    ssh hadoop102 "/opt/module/hadoop/sbin/stop-dfs.sh"
    ;;
*)
    echo "用法: $0 {start|stop}"
    ;;
esac
```

---

## 第3章 常用面试题 🔥

### Q1：如何查看某个端口是否被占用？
> `netstat -nltp | grep 端口号` 或 `lsof -i:端口号`

### Q2：如何实时查看日志？
> `tail -f 日志文件路径`

### Q3：如何在集群中分发文件？
> 使用 `rsync` 或 `scp`，编写 xsync 脚本批量分发。
