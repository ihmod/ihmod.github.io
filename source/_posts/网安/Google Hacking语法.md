---
title: Google Hacking 语法
tags:
  - google hacking
categories: 网安
top_img: false
description: Google Hacking 语法
abbrlink: f8b0d770
---
> 可以自行选择payload生成社工字典，使用工具跑一跑

- 查找语法示例

  intext:管理
  filetype:mdb
  inurl:file
  site:xx.com filetype:txt  （查找TXT文件 其他的依次内推）
  site:xx.com intext:管理
  site:xx.com inurl:login
  site:xx.com intitle:后台
  
  
  
- 查找某个网站的后台地址
   site:xxx.com intext:管理|后台|登录|登陆|用户名|密码|系统|账号|login|system|admin
   inurl:edu.cn intitle:管理
   site:xxx.com inurl:login|inurl:admin|inurl:admin_login|inurl:system
   site:xxx.com intitle:管理|后台|后台管理|登录|登陆
   inurl:login|admin|admin_login|login_admin|system|user
   site:xxx.com

- 查找文本内容
   site:xxx.com intext:管理|后台|登录|用户名|密码|验证码|系统|admin|login|username|password

- 查找可注入点
   site:xxx.com inurl:aspx|jsp|php|asp
   site:xxx.com inurl:php?id=

- 查找文件上传漏洞
   site:xxx.com inurl:file|load|editor|files|

- 查找eweb编辑器
   site:xxx.com inurl:ewebeditor|editor|uploadfile|eweb|edit

- 查找存在的数据库
   site:xxx.com filetype:mdb
   site:xxx.com filetype:数据库格式

- 查看脚本类型
   site:xxx.com filetype:asp/aspx/php/jsp
   site:xxx.com filetype:php

- 查找目录遍历漏洞
   site:xxx.com intitle:index of 

- 社工信息
   site:xxx.com intitle:账号|密码|工号|学号|身份证

- 搜索各类开源的网站上面的信息
   site:github.com intext:xiaodi8.com

- 实战演示：
  首先用google搜索这个站点的基本情况

   site:xxx.com
  从搜索结果中找到了几处该站点的域名

   http://aaa.xxx.com
   http://bbb.xxx.com
   http://ccc.xxx.com
  然后查看这几个域名的ip，并确认是否存在CDN

  发现不存在CDN服务，并且子域名ip有的也不同

  然后搜索该站点的后台地址

   site:xxx.com intext:管理
   site:xxx.com inurl:login
   site:xxx.com inurl:admin
   site:xxx.com intitle:管理
   等其他方式也可以，自行组合即可
  最终获取到了多个后台地址

  然后查看服务器脚本语言

  site:aaa.xxx.com filetype:asp
   site:aaa.xxx.com filetype:php
   site:aaa.xxx.com filetype:aspx
   site:aaa.xxx.com filetype:jsp
   等其他方式也可判断
  获取了基本的搭建组合后

  下一步就尝试获取漏洞了

  site:aaa.xxx.com intext:ftp://*.*
   site:bbb.xxx.com inurl:file
   site:ccc.xxx.com inurl:load
   等其他漏洞的关键字自行搜索
  得到地址后就可以进行下一步渗透了
  
  

- 获取二级域名
   site:xxx.com 

-  获取邮箱地址
   site:xxx.com intext:*@xxx.com

- 获取电话信息
   site:xxx.com intext:电话


- 查看服务器使用的程序
  site:xx.com filetype:asp
  site:xx.com filetype:php
  site:xx.com filetype:jsp

- 查看上传漏洞：
  site:xx.com inurl:file
  site:xx.com inurl:load

- 查找注入点：
  site:xx.com filetype:asp
  site:tw inurl:asp?id= 这个是找台湾的
  site:jp inurl:asp?id= 这个是找日本的
  site:ko inurl:asp?id= 这个是找韩国的
  依次类推
  intitle:旁注- 网站xxxfiletype:asp
  inurl:editor/db/
  inurl:eWebEditor/db/
  inurl:bbs/data/
  inurl:databackup/
  inurl:blog/data/
  inurl:okedata
  inurl:bbs/database/
  inurl:conn.asp
  inc/conn.asp

- 管理入口:
  admin
  admin_index
  admin_admin
  index_admin
  admin/index
  admin/default
  admin/manage
  admin/login
  manage_index
  index_manage
  superadmin
  说明.txt
  manager/login
  manager/login.asp
  manager/admin.asp
  login/admin/admin.asp
  houtai/admin.asp
  guanli/admin.asp
  denglu/admin.asp
  admin_login/admin.asp
  admin_login/login.asp
  admin/manage/admin.asp
  admin/manage/login.asp
  admin/default/admin.asp
  admin/default/login.asp
  member/admin.asp
  member/login.asp
  administrator/admin.asp
  administrator/login.asp

- 强大的数据库:
  database/PowerEasy4.mdb #动易网站管理系统4.03数据库
  database/PowerEasy5.mdb
  database/PowerEasy6.mdb
  database/PowerEasy2005.mdb
  database/PowerEasy2006.mdb
  database/PE_Region.mdb
  data/dvbbs7.mdb #动网论坛数据库
  databackup/dvbbs7.mdb #动网论坛备份数据库
  bbs/databackup/dvbbs7.mdb #动网论坛备份数据库
  data/zm_marry.asp #动网sp2美化版数据库
  databackup/dvbbs7.mdb
  admin/data/qcdn_news.mdb #青创文章管理系统数据库
  firend.mdb #交友中心数据库
  database/newcloud6.mdb #新云管理系统6.0数据库
  database/%23newasp.mdb #新云网站系统
  blogdata/L-BLOG.mdb #L-BLOG v1.08数据库
  blog/blogdata/L-BLOG.mdb #L-BLOG v1.08数据库
  database/bbsxp.mdb #BBSXP论坛数据库
  bbs/database/bbsxp.mdb #BBSXP论坛数据库
  access/sf2.mdb #雪人论坛程序v2.0数据库
  data/Leadbbs.mdb #LeadBBS论坛 v3.14数据库
  bbs/Data/LeadBBS.mdb #LeadBBS论坛 v3.14数据库
  bbs/access/sf2.mdb #雪人论坛程序v2.0数据库
  fdnews.asp #六合专用BBS数据库
  bbs/fdnews.asp #六合专用BBS数据库
  admin/ydxzdate.asa #雨点下载系统 v2.0+sp1数据库
  data/down.mdb #动感下载系统xp ver2.0数据库
  data/db1.mdb #动感下载系统xp v1.3数据库
  database/Database.mdb #轩溪下载系统 v3.1数据库
  db/xzjddown.mdb #lhdownxp下载系统数据库
  db/play.asp #娱乐先锋论坛 v3.0数据库
  mdb.asp #惊云下载系统 v1.2数据库
  admin/data/user.asp #惊云下载系统 v3.0数据库
  data_jk/joekoe_data.asp #乔客6.0数据库
  data/news3000.asp #沸腾展望新闻系统 v1.1数据库
  data/appoen.mdb #惠信新闻系统4.0数据库
  data/12912.asp #飞龙文章管理系统 v2.1数据库
  database.asp #动感极品下载管理系统 v3.5
  download.mdb #华仔软件下载管理系统 v2.3
  dxxobbs/mdb/dxxobbs.mdb #dxxobbs论坛数据库
  db/6k.asp #6kbbs 用户名:admin 密码:6kadmin
  database/snowboy.mdb #雪孩论坛 默认后台admin/admin_index.asp
  database/%23mmdata.mdb #依爽社区
  editor/db/ewebeditor.mdbeWebEditor/db/ewebeditor.mdb

- 上传漏洞:
  /eWebEditor/upload.asp #eWebEditor上传页面
  /editor/upload.asp #eWebEditor上传页面
  /bbs/upfile.asp #动网论坛上传页面
  /forum/upfile.asp #动网论坛上传页面
  /dvbbs/upfile.asp #动网论坛上传页面
  /upfile_soft.asp #动力管理系统上传页面
  /upload.asp?action=upfile #乔客6.0上传页面
  /upfile.asp #动网论坛上传页面
  /bbs/down_addsoft.asp #动网论坛插件上传页面
  /bbs/down_picupfile.asp #动网论坛插件上传页面
  /down_picupload.asp #动网论坛插件上传页面
  /admin/admin_upfile.asp #管理员后台上传页面
  /admin/upfile.asp #管理员后台上传页面
  /admin/upload.asp #管理员后台上传页面
  /admin/uploadfaceok.asp #尘缘上传页面
  /news/admin/upfile.asp #新闻管理上传页面
  /admin_upfile.asp #飞龙文章管理系统 v2.0
  /user_upfile.asp #飞龙文章管理系统 v2.0
  /upload_flash.asp #秋叶购物商城上传页面
  /Saveannounce_upload.asp #购物中心上传页面
  /UploadFace.asp #沸腾展望新闻系统 v1.1
  /bbs/diy.asp #Domian3.0默认木马
  /UploadSoft/diy.asp #Domian3.0默认木马
  /diy.asp #Domian3.0默认木马
  /upload/upload.asp #某某文章管理系统
  /mybbs/saveup.asp #MYBBS论坛上传页面
  /dxxobbs/upload.asp #DxxoBBS论坛上传页面
  /img_upfile.asp #任我飞扬驿站上传页面
  /Upfile_SoftPic.asp #动力管理系统上传页面
  /upfile_flash.asp #秋叶购物商城上传页面

- PHP的:
  phpmyadmin
  include/config.inc.php
  include/config.php
  lib/config.php

- inurl中的管理后台路径
  nurl:admin/manager
  admin
  admin_index
  admin_admin
  index_admin
  admin/index
  admin/default
  admin/manage
  admin/login
  manage_index
  index_manage
  manager/login
  manager/login.asp
  manager/admin.asp
  login/admin/admin.asp
  houtai/admin.asp
  guanli/admin.asp
  denglu/admin.asp
  admin_login/admin.asp
  admin_login/login.asp
  admin/manage/admin.asp
  admin/manage/login.asp
  admin/default/admin.asp
  admin/default/login.asp
  member/admin.asp
  member/login.asp
  administrator/admin.asp
  administrator/login.asp

- inurl中的数据库和配置文件
  inurl:editor/db/
  inurl:eWebEditor/db/
  inurl:bbs/data/
  inurl:databackup/
  inurl:blog/data/
  inurl:okedata
  inurl:bbs/database/
  inurl:conn.asp
  inurl:inc/conn.asp
  inurl:“viewerframe?mode=”//搜索摄像头地址
  inurl:db
  inurl:mdb
  inurl:config.txt
  inurl:bash_history
  inurl:data filetype:mdb //搜索mdb格式数据库

- inurl中的搜索备份文件
  inurl:temp
  inurl:tmp
  inurl:backup
  inurl:bak

- inurl中查找注入点
  site:xx.com filetype:asp
  site:tw inurl:asp?id= //这个是找台湾的
  site:jp inurl:php?id= // 这个是找日本的
  site:ko inurl:jsp?id= // 这个是找韩国的

- inurl之上传漏洞
  site:xx.com inurl:file
  site:xx.com inurl:load

- 目标遍历漏洞
  Index of /admin
  Index of /passwd
  Index of /password
  Index of /mail
  “Index of /” +passwd
  “Index of /” +password.txt
  “Index of /” +.htaccess
  “Index of /secret”
  “Index of /confidential”
  “Index of /root”
  “Index of /cgi-bin”
  “Index of /credit-card”
  “Index of /logs”
  “Index of /config”
  “indexof/”inurl:lib //搜索图书馆一般目录遍历下载

- 目录遍历及敏感信息泄露
  intitle:“index of” etc
  intitle:“Index of” .sh_history
  intitle:“Index of” .bash_history
  intitle:“index of” passwd
  intitle:“index of” people.lst
  intitle:“index of” pwd.db
  intitle:“index of” etc/shadow
  intitle:“index of” spwd
  intitle:“index of” master.passwd
  intitle:“index of” htpasswd
  intitle:“index of” admin
  inurl:service.pwd
  intitle:phpmyadmin intext:Create new database //搜索phpmyadmin直接进入后台
  intitle:“php shell*” “Enable stderr” filetype:php//批量搜索webshell
  intitle:“index of” data //列出data目录
  intilte:“error occurred” ODBC request where (select|insert) //搜索sql注入出错页面
  intitle:index.of filetype:log //搜索日志文件



- tomcat
  如果使用暴力破解成功突破tomcat的basic认证，上传恶意war包，可以很轻易的getshell
  #site用于限制查询范围,intitle和intext用于匹配tomcat关键字
  intitle:apache tomcat site:domain

- weblogic
  weblogic框架曾爆出不少漏洞，如果存在该框架，是一个突破口
  #site用于限制查询范围,inurl和intitle用于匹配weblogic的关键字
  inurl:/console/login/LoginForm.jsp site:domain
  inurl:/console/login/LoginForm.jsp intitle:Oracle WebLogic Server site:domain
  inurl:/console/login/ intitle:“Oracle WebLogic Server 管理控制台” site:domain

- jboss
  jboss框架曾爆出不少漏洞，如果存在该框架，是一个突破口
  #site用于限制查询范围,inurl用于匹配jboss的关键字
  inurl:/jmx-console/htmladaptor site:domain

- websphere
  websphere框架曾爆出不少漏洞，如果存在该框架，是一个突破口
  #site用于限制查询范围,inurl用于匹配websphere的关键字
  inurl:/ibm/console/logon.jsp site:domain

- phpmyadmin
  phpmyadmin是MySQL的管理平台，并且可被爆破，一旦爆破成功，即可获取MySQL的权限，进而提权getshell
  #site用于限定范围,inurl和intext用于匹配phpMyAdmin的关键字
  inurl:/phpMyAdmin site:domain
  inurl:/phpMyAdmin/index.php site:domain
  inurl:/phpMyAdmin/index.php site:domain db+information_schema
  inurl:/phpMyAdmin/index.php intext:phpMyAdmin site:domain

- webmin
  webmin是一个web版的linux系统管理工具，默认情况下工作在web的10000端口上
  #site用于限制查询范围 intitle和intext用于匹配webmin的关键字
  intitle:Login to Webmin intext:“login to the Webmin server on” site:domain

- wordpress
  wordpress是PHP的开源博客平台，其框架存在大量漏洞，尤其其框架存在很多SQL注入，通过SQL注入getshell
  #site用于限制查询范围,inurl和index of用于匹配wordpress的关键字
  inurl:/wp-login.php site:domain
  index of /wp-content/uploads inurl:/wp-login.php site:domain
  inurl:/wp-content/themes/theagency site:domain

- joomla
  joomla框架曾爆出不少漏洞，如果存在该框架，是一个突破口
  #site用于限制查询范围,inurl用于匹配joomla的关键字
  inurl:/administrator/index.php site:domain
  inurl:index.php?option=com_advertisementboard site:domain 找注入
  inurl:index.php?option=com_carocci site:domain
  inurl:index.php?option=com_product site:domain
  inurl:/administrator/index.php site:domain

- drupal
  drupal框架曾爆出不少漏洞，如果存在该框架，是一个突破口
  #site用于限制查询范围,inurl和intext用于匹配drupal的关键字
  inurl:CHANGELOG.txt intext:drupal intext:“SA-CORE” -site:github.com -site:domain

- 特征查找
  个人觉得这个不错
  #在使用时最好加上site限制查询范围,且版本号也可忽略
  power by wordpress powered by discuz x3.2
  powered by phpcms 2008 powered by drupal 7
  powered by dedecmsv57_gbk powered by CubeCart 3.0.6
  Powered by phpBB 2.0.6 powered by paBugs 2.0 Beta 3
  inurl:wp-login.php inurl:/administrator/index.php
  inurl:/admina.php

- owa
  outlook邮箱
  inurl:/owa/auth/logon.aspx site:domain

- vpn
  inurl:/sslvpn site:domain
  mirapoint

- 自行添加site以限制查询范围
  inurl:/cgi-bin/search.cgi site:domain
  inurl:/cgi-bin/madmin.cgi site:domain

- zimbra
  自行添加site以限制查询范围
  inurl:7071/zimbraAdmin/ site:domain
  inurl:/help/en_US/standard/version.htm site:domain

- 常见的后台地址
  有用 ！！ 自行添加site以限制查询范围
  inurl:/manager/login.php site:domain
  inurl:/cms/login.php site:domain
  inurl:/manage/index.php site:domain
  inurl:/system/login.php site:domain
  inurl:/webadmin/login.php site:domain
  inurl:admin_login.php intitle:admin login site:domain
  inurl:admin_login.php intitle:admin page site:domain
  inurl:/admin/login.php site:domain
  inurl:/admin/index.php site:domain
  inurl:/system/adminlogin.asp site:domain
  inurl:/manage/login.aspx site:domain
  inurl:/sysadm/index.php site:domain
  intext:“Website Design & Developed By : WebSay” 默认后台/admin
  intext:“Powered by ENS Consultants” 默认后台/admin/login.php
  intext:“Desenvolvimento - MW Way” 默认后台/admin/index.php
  inurl:.php?id= intext:“Web realizada por Soma Estudio”
  inurl:/_mycps/login.php
  intext:“design by weli” 默认后台: /adm/login.php 除了弱口令还有注入(linjizen@gmail.com/lin719192)
  inurl:categorysearch.php?indus= site:domain SQL注入

- svn
  自行添加site以限制查询范围
  inurl:/.svn/entries site:domain

- 上传点
  自行添加site以限制查询范围
  intext:" Powered by JADBM " JADBM Cms upload shell 注册后登陆上传即可
  inurl:"/index.php/frontend/login/en" Estate cms upload shell 注册后登陆上传即可
  inurl:/Content/Roxy_Fileman/ 该路径下直接就是上传点
  index of:“filemanager/dialog.php” 该脚本就是上传脚本直接上传即可
  intext:“Desenvolvido por Webnet Soluções Tecnológicas.” fck上传
  inurl:“subir_foto.php” 上传点
  inrul:"/imce?dir=" intitle:“File Browser”
  inurl:“Powered by Vision Helpdesk 3.9.10 Stable” 注册后登陆进去编辑个人配置上传
  index of /admin/fckeditor site:.tw
  inurl:/ewebeditor/ site:.tw
  inurl:/admin/upload_file.php
  inurl:/admin/upfile.php
  inurl:/admin/upload.asp

- 文件包含和命令执行
  自行添加site以限制查询范围
  inurl:footer.inc.php?settings=
  inurl:/pb_inc/admincenter/index.php?page=
  inurl:/pnadmin/categories.inc.php?subpage=
  inurl:/index.php??view=src/sistema/vistas/
  inurl:/edit.php?em=file&filename=
  inurl:/path_to_athena/athena.php?athena_dir= 远程包含
  inurl:/path_to_qnews/q-news.php?id= 远程包含
  inurl:/inc/backend_settings.php?cmd=
  inurl:login.action strus2系列执行漏洞利用
  inurl:php?x= inurl:php?open=
  inurl:php?visualizar= inurl:php?pagina=
  inurl:php?inc= inurl:php?include_file=
  inurl:php?page= inurl:php?pg=
  inurl:php?show= inurl:php?cat=
  inurl:php?file= inurl:php?path_local=
  inurl:php?filnavn= inurl:php?HCL_path=
  inurl:php?doc= inurl:php?appdir=
  inurl:php?phpbb_root_dir= inurl:php?phpc_root_path=
  inurl:php?path_pre= inurl:php?nic=
  inurl:php?sec= inurl:php?content=
  inurl:php?link= inurl:php?filename=
  inurl:php?dir= inurl:php?document=
  inurl:index.php?view= inurl:.php?locate=
  inurl:.php?place= inurl:.php?layout=
  inurl:.php?go= inurl:.php?catch=
  inurl:.php?mode= inurl:.php?name=
  inurl:.php?loc= inurl:.php?f=
  inurl:.php?inf= inurl:.php?pg=
  inurl:.php?load= inurl:.php?naam=
  allinurl:php?page= allinurl:php?file=
  inurl:php?x= inurl:admin.php?cal_dir=
  inurl:php?include= inurl:php?nav=
  inurl:.php?sel= inurl:php?p=
  inurl:php?conf= inurl:php?prefix=
  inurl:theme.php?THEME_DIR=
  inurl:php?lvc_include_dir=
  inurl:php?basepath= inurl:php?pm_path=
  inurl:php?user_inc= inurl:php?cutepath=
  inurl:php?fil_config= inurl:php?libpach=
  inurl:php?pivot_path= inurl:php?rep=
  inurl:php?conteudo= inurl:php?root=
  inurl:php?configFile inurl:php?pageurl
  inurl:php?inter_url inurl:php?url=
  inurl:php?cmd= inurl:path.php?my=
  inurl:php?xlink= inurl:php?to=
  inurl:file.php?disp=

- 商城类
  自行添加site以限制查询范围
  inurl:".php?catid=" intext:“View cart”
  inurl:".php?catid=" intext:“Buy Now”
  inurl:".php?catid=" intext:“add to cart”
  inurl:".php?catid=" intext:“shopping”
  inurl:".php?catid=" intext:“boutique”
  inurl:".php?catid=" intext:"/store/"
  inurl:".php?catid=" intext:"/shop/"
  inurl:".php?catid=" intext:“Toys”
  inurl:details.php?BookID=
  inurl:shop.php?do=part&id=

- CMS
  自行添加site以限制查询范围
  inurl:article.php?ID= inurl:newsDetail.php?id=
  inurl:show.php?id= inurl:newsone.php?id=
  inurl:news.php?id= inurl:event.php?id=
  inurl:preview.php?id= inurl:pages.php?id=
  inurl:main.php?id= inurl:prod_detail.php?id=
  inurl:view.php?id= inurl:product.php?id=
  inurl:contact.php?Id= inurl:display_item.php?id=
  inurl:item.php?id= inurl:view_items.php?id=
  inurl:details.asp?id= inurl:profile.asp?id=
  inurl:content.asp?id= inurl:display_item.asp?id=
  inurl:view_detail.asp?ID= inurl:section.php?id=
  inurl:theme.php?id= inurl:produit.php?id=
  inurl:chappies.php?id= inurl:readnews.php?id=
  inurl:rub.php?idr= inurl:pop.php?id=
  inurl:person.php?id= inurl:read.php?id=
  inurl:reagir.php?num= inurl:staff_id=
  inurl:gallery.php?id= inurl:humor.php?id=
  inurl:spr.php?id= inurl:gery.php?id=
  inurl:profile_view.php?id=
  inurl:fellows.php?id= inurl:ray.php?id=
  inurl:productinfo.php?id=
  inurl:file.php?cont= inurl:include.php?chapter=
  inurl:principal.php?param=
  inurl:general.php?menue= inurl:php?pref=
  inurl:nota.php?chapter= inurl:php?str=
  inurl:php?corpo= inurl:press.php?[]*=
  inurl:asp?pid= inurl:php?id=
  inurl:aspx?id= inurl:jsp?id=
  inurl:do?id= inurl:cgi?id=

- 万能密码
  自行添加site以限制查询范围
  inurl:“wladmin/login.asp”
  Username : ‘=’ ‘or’
  Password : ‘=’ ‘or’
  intext:POWERED BY Versatile Software Services 默认后台/alogin.aspx
  User ==> ‘or’’=’
  Pass ==> ‘or’’=’
  inurl:/media.php?hal=login
  Email: ‘=’‘or’@gmail.com
  Pass: ‘=’‘or’
  intext:“Powered by : Best Webmasterz.” 默认后台/admin
  User : ‘=’ ‘OR’
  Pass : ‘=’ ‘OR’
  intext:“Web Design and Maintenance by Cloud 5 Solutions” 默认后台/admin/login.php
  User : ‘=’ ‘OR’
  Pass : ‘=’ ‘OR’
  intext:“网站设计：火龙科技” 默认后台/maintain/login.php
  Username : ‘=’ ‘or’
  Password : ‘=’ ‘or’
  intext:“Powered by Moodyworld” 默认后台/admin/
  Username : ‘=’ ‘or’
  Password : ‘=’ ‘or’

- 敏感信息泄露
  自行添加site以限制查询范围
  site:domain inurl:/phpinfo.php
  filetype:log “PHP Parse error”| “PHP Warning”
  site:domain “id=” & intext:"Warning: mysql_fetch_array()
  site:domain “id=” & intext:"Warning: getimagesize()
  site:domain “id=” & intext:"Warning: array_merge()
  site:domain “id=” & intext:"Warning: mysql_fetch_assoc()
  site:domain “id=” & intext:"Warning: mysql_result()
  site:domain “id=” & intext:"Warning: pg_exec()
  site:domain “id=” & intext:"Warning: require()
  inurl:/robots.txt site:.
  inurl:/application/configs/ 配置文件名为/application/configs/application.ini
  ----------------------------htpasswd--------------------------------
  htpasswd.bak filetype:htpasswd
  -----------------------------cisco vpn----------------------------
  filetype:pcf “GroupPwd”
  cisco在线密码解密网站==>https://www.unix-ag.uni-kl.de/~massar/bin/cisco-decode
  -----------------ftp 账号密码-----------------------------
  “index of/” “ws_ftp.ini” “parent directory”
  “your password is” filetype:log
  filetype:ini inurl:“serv-u.ini”
  filetype:ini inurl:flashFXP.ini
  filetype:ini ServUDaemon
  filetype:ini wcx_ftp
  filetype:ini ws_ftp pwd
  ext:inc “pwd=” “UID=”
  auth_user_file.txt
  filetype:sql inurl:backup inurl:wp-content
  inurl:/eWebEditor/db/ site:domain
  filetype:xls QQ site:cn

- 目录遍历
  site:domain index of /admin
  site:domain index of /upfiles
  site:domain index of /fckeditor/editor/
  site:domain index of /admin/uploadfile
  site:domain index of /admin/file
  site:domain index of /system/file
  site:domain index of /phpmyadmin
  site:domain index of /web/backup/
  inurl:/phpmyadmin/index.php site:domain

- 遗留webshell
  自行添加site以限制查询范围
  inurl:b374k.php filetype:php
  inurl:c99.php
  inurl:c100.php Generation time:
  inurl:itsecteam_shell.php
  intext:x2300 Locus7Shell v. 1.0a beta Modded by
  intext:c99shell inurl:c99.php
  powered by Captain Crunch Security Team
  “inurl:c99.php” + “intext:safe”
  intitle:r57shell
  intitle:c99shell +uname
  inurl:c99.php uid=0(root)
  intitle:c99shell+filetype:php
  intitle:ly0kha shell
  inurl:.php “cURL: ON MySQL: ON MSSQL: OFF”
  “Shell” filetypehp intext:“uname -a:” “EDT 2010”
  intitle:“intitle:r57shell”
  inurl:“c99.php” & intext:Encoder Tools Proc.
  inurl:“c100.php” & intext:Encoder Tools Proc.
  intitle:“Shell” inurl:".php" & intext:Encoder Tools Proc.