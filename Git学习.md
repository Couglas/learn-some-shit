# github创建仓库并连接本地

1. github
   1. login
   2. new repository -> fill some shit -> https://github.com/username/learn-some-shit.git
   3. Settings -> Developer Settings -> Personal access tokens -> Tokens (classic) -> Generate new token -> select some shit -> token
2. local
   1. mkdir learn-some-shit
   2. cd learn-some-shit
   3. git init
   4. gr add origin https://github.com/username/learn-some-shit.git
   5. grset origin https://token@github.com/username/learn-some-shit.git
   8. gl origin master 
   9. vim someshit.txt
   10. gst, gaa, gcmsg 'add some shit', gp

# 常用命令

常用命令别名参考此地址：https://kapeli.com/cheat_sheets/Oh-My-Zsh_Git.docset/Contents/Resources/Documents/index

gl

gp

gco, gcb

gf

gb, gba, gbr,  gbd, gbD

gsta, gstp, gstd, gstl, gstc

gst

ga, gaa

gcmsg ''

gcf

gcp, gcpa, gcpc

gd

ghh

glg, glog, glgg, gloga

gm

gr, gra

grb, grbi,  grba, grbc

grh, grhh

grm

gsh