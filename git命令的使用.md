
1. git init //初始化仓库

2. git add .(文件name) //添加文件到本地仓库

3. git commit -m "first commit" //添加文件描述信息 这个描述信息是必须的

4. git remote add origin git@github.com:yuxhe896155/Thoughts.gi //链接远程仓库，创建主分支

5. git pull origin master // 把本地仓库的变化连接到远程仓库主分支

6. git push -u origin master //把本地仓库的文件推送到远程仓库


注意出现问题需要强制合并哈
git pull origin master --allow-unrelated-histories

拉仓库下的wiki方法
git  clone git@github.com:l1966540314/iOSUnderNotes.wiki.git

强制覆盖本地
git reset --hard origin/master

-----------------------------------------拉某子目录
git init mstm
cd mstm
git config core.sparsecheckout true 
echo 'books/面试题/*' >> .git/info/sparse-checkout 
git remote add origin git@github.com:JJSilence/books.git  
git pull origin master
----------------------------拉某子目录
git clone -n https://github.com/tensorflow/models
cd tensorflow
git config core.sparsecheckout true
echo official/resnet/* >> .git/info/sparse-checkout
git checkout master
-----------------------------------
