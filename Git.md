# Git

## 初始化一个项目并上传到GitHub
```
ssh-keygen -t rsa -C "geyichn@126.com" -f github
ssh-agent bash
ssh-add github
cat github.pub
```

```
git config --global user.email "geyichn@126.com"
git config --global user.name "geyi"
git init
git add <file>
git commit -m ''
git remote add origin git@github.com:geyi/ReliableGold.git
git push --set-upstream origin master
```

## Git Bash中文乱码问题
1. Options -> text -> Local & Character set
2. 执行 `git config --global core.quotepath false`
3. 使用UTF-8字符集编码保持文件，否则 `git diff` 会出现乱码