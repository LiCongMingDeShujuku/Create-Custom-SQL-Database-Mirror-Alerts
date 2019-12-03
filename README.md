![CLEVER DATA GIT REPO](https://raw.githubusercontent.com/LiCongMingDeShujuku/git-resources/master/0-clever-data-github.png "李聪明的数据库")

# 创建自定义SQL数据库镜像警报
#### Create Custom SQL Database Mirror Alerts
**发布-日期: 2015年9月18日 (评论)**

## Contents

- [中文](#中文)
- [English](#English)
- [SQL Logic](#Logic)
- [Build Info](#Build-Info)
- [Author](#Author)
- [License](#License) 


## 中文
以下是一些你可以用在任何作业上的sql逻辑（logic）。创建数据库镜像警报程序。作业旨在检查所有数据库（不包括系统级别），以确保它们已根据其Create_Date或Restore_Date（最新的）配置为镜像。它不检查镜像数据库。只检查Principals。 这可以部署在当前被配置使用数据库镜像的任何SQL Server实例中。在Primary或Secondary上部署是安全的。如果尚未配置数据库，则会编译一个所述数据库的创建/恢复日期的列表，并发送一封电子邮件。此外，
它只会检查在其上配置了Principal的数据库实例，因此如果你想在Mirror上创建此作业，请随意。在将数据库配置为服务器上的Principal之前，它不会执行任何操作。因此，只要数据库实例通过“set partner failover”进行故障转移，
作业就会开始运行。
下面是收到的邮件的样子：

## English
Here’s some sql logic you can throw into any Job. Create Database Mirror Alert Process. Job is designed to check all databases (excluding system level) to ensure they have been configured for Mirroring based on their Create_Date or Restore_Date (which is most recent). It does NOT check Mirror Databases. Only Principals. This can be deployed across any SQL Server Instance that is currently configured to use Database Mirroring. It is safe to deploy on both Primary or Secondary. If databases have not been configured it compiles a list along with the create/restore dates of said databases, and sends an email. Additionally it will ONLY check Database Instances that have Principals configured on them so if you want to create this Job on the Mirror feel free. It won’t do anything until a database has been configured as the Principal on the Server. So whenever the Database Instance fails over via ‘set partner failover’, then the Job will start running.

Here is what the email will look like:

![#](images/01-Create-Custom-SQL-Database-Mirror-Alerts.png?raw=true "#")

Here is what the Job will do exactly.
1. Check to see if SQL Database Mail has been configured.
2. Configures SQL Database Mail (automatically) if it does not find the configuration. You will need to supply the SMTP Server Name below.
3. Sends a test email once SQL Database Mail has been configured.
4. Checks to see if ALL databases (excluding system databases) has been configured for SQL Database Mirroring.
5. Creates a temporary table to hold database Names, Create_Date, and Restore_Date.
6. Emails Database List.

以下是作业将要做的事情。
1. 检查是否已配置SQL数据库邮件。
2. 如果找不到配置，则自动配置SQL数据库邮件。 需要你在下面提供SMTP服务器名称。
3. 配置SQL数据库邮件后发送一封测试电子邮件。
4. 检查是否已为SQL数据库镜像配置了所有数据库（不包括系统数据库）。
5. 创建一个临时表来保存数据库的Names，Create_Date和Restore_Date。
6. 发送数据库列表电子邮件。


---
## Logic
```SQL
use msdb;
set nocount on
set ansi_nulls on
set quoted_identifier on
 
----------------------------------------------------------------------
-- 	Configure SQL Database Mail if it's not already configured.
-- 	配置SQL数据库邮件（如果尚未配置）。
if (select top 1 name from msdb..sysmail_profile) is null
    begin
        ----------------------------------------------------------------------
-- 	Enable SQL Database Mail
-- 	启用SQL数据库邮件
        exec master..sp_configure 'show advanced options',1
        reconfigure;
        exec master..sp_configure 'database mail xps',1
        reconfigure;
 
        ----------------------------------------------------------------------
-- 	Add a profile
-- 	添加文件
        execute msdb.dbo.sysmail_add_profile_sp
            @profile_name       = 'SQLDatabaseMailProfile'
        ,   @description        = 'SQLDatabaseMail';
 
        ----------------------------------------------------------------------
-- 	Add the account names you want to appear in the email message.
-- 	添加要在电子邮件中显示的帐户名称。
        execute msdb.dbo.sysmail_add_account_sp
            @account_name       = 'sqldatabasemail@mydomain.com'
        ,   @email_address      = 'sqldatabasemail@mydomain.com'
        ,   @mailserver_name    = 'MySMTPServerName@MyDomain.com  
        --, @port           = ####  --optional 可选择的
        --, @enable_ssl     = 1 –optional 可选择的
        --, @username       ='MySQLDatabaseMailProfile' --optional
        --, @password       ='MyPassword' --optional
 
-- 	Adding the account to the profile
-- 	将帐户添加到文件中
        execute msdb.dbo.sysmail_add_profileaccount_sp
            @profile_name       = 'SQLDatabaseMailProfile'
        ,   @account_name       = 'sqldatabasemail@mydomain.com'
        ,   @sequence_number    = 1;
 
-- 	Give access to new database mail profile (DatabaseMailUserRole)
-- 	授予对新数据库邮件（new database mail）文件的访问
        execute msdb.dbo.sysmail_add_principalprofile_sp
            @profile_name       = 'SQLDatabaseMailProfile'
        ,   @principal_id       = 0
        ,   @is_default     = 1;
 
        ----------------------------------------------------------------------
-- 	Get Server info for test message
-- 	测试获取服务器信息
 
        declare @get_basic_server_name              varchar(255)
        declare @get_basic_server_name_and_instance_name        varchar(255)
        declare @basic_test_subject_message             varchar(255)
        declare @basic_test_body_message                varchar(max)
        set @get_basic_server_name              = (select cast(serverproperty('servername') as varchar(255)))
        set @get_basic_server_name_and_instance_name        = (select  replace(cast(serverproperty('servername') as varchar(255)), '\', '   SQL Instance: '))
        set @basic_test_subject_message             = 'Test SMTP email from SQL Server: ' + @get_basic_server_name_and_instance_name
        set @basic_test_body_message                = 'This is a test SMTP email from SQL Server:  ' + @get_basic_server_name_and_instance_name + char(10) + char(10) + 'If you see this.  It''s working perfectly :)'
 
        ----------------------------------------------------------------------
-- 	Send quick email to confirm email is properly working.
-- 	发送快速电子邮件确认电子邮件是否正常工作
 
        EXEC msdb.dbo.sp_send_dbmail
            @profile_name       = 'SQLDatabaseMailProfile'
        ,   @recipients     = 'SQLJobAlerts@mydomain.com'
        ,   @subject        = @basic_test_subject_message
        ,   @body           = @basic_test_body_message;
 
-- 	Confirm message send
-- 	确认邮件发送成功
        -- select * from msdb..sysmail_allitems
    end
 
----------------------------------------------------------------------------------------
-- 	get basic server info.
-- 	获取基本的服务器信息
 
declare @server_name_basic          varchar(255)
declare @server_name_instance_name      varchar(255)
declare @server_time_zone           varchar(255)
set @server_name_basic          = (select cast(serverproperty('servername') as varchar(255)))
set @server_name_instance_name      = (select  replace(cast(serverproperty('servername') as varchar(255)), '\', '   SQL Instance: '))
 
----------------------------------------------------------------------------------------
-- 	set message subject.
-- 	设置短信对象
declare @message_subject            varchar(255)
set @message_subject            = 'Server: ' + @server_name_instance_name + ' missing Mirror configuration.'
 
----------------------------------------------------------------------------------------
----------------------------------------------------------------------------------------
----------------------------------------------------------------------------------------
----------------------------------------------------------------------------------------
----------------------------------------------------------------------------------------
-- 	get databases that have NOT been configured for Database Mirroring
-- 	获取尚未配置数据库镜像的数据库
if object_id('tempdb..#created_or_restored') is not null
    drop table #created_or_restored
 
create table #created_or_restored
    (
        [database]  varchar(255)
    ,   [created_on]    varchar(255)
    ,   [restored_on]   varchar(255)
    );
with last_restored as
(
    select
    databasename = sd.[name]
,   sd.[create_date]
 
,   rh.*
,   rownum = row_number() over (partition by sd.name order by rh.[restore_date] desc)
 
    from
    master.sys.databases sd left outer join msdb.dbo.[restorehistory] rh on rh.[destination_database_name] = sd.name
    join master.sys.database_mirroring sdm on sd.database_id = sdm.database_id
    where
    sd.database_id > 4
    and sd.state_desc = 'online'
    and sdm.mirroring_role_desc is null
    --or    sdm.mirroring_role_desc != 'mirror'
)
insert into #created_or_restored
select
    'database'      = upper(databasename)
,   'created_on'        = replace(replace(left(create_date, 19), 'AM', 'am'), 'PM', 'pm') + ' ' + datename(dw, create_date)
,   'restored_on'       = 
                    case
                        when restore_date is not null then replace(replace(left(restore_date, 19), 'AM', 'am'), 'PM', 'pm') + ' ' + datename(dw, create_date)
                        else ''
                    end
from
    [last_restored]
where
    [rownum] = 1
    and databasename not in ('master', 'model', 'msdb', 'tempdb')
----------------------------------------------------------------------------------------
----------------------------------------------------------------------------------------
----------------------------------------------------------------------------------------
----------------------------------------------------------------------------------------
----------------------------------------------------------------------------------------
-- 	create conditions for html tables in top and mid sections of email.
-- 	在电子邮件的顶部和中间部分为html表格创建条件。
 
declare @xml_top            NVARCHAR(MAX)
declare @xml_mid            NVARCHAR(MAX)
declare @body_top           NVARCHAR(MAX)
declare @body_mid           NVARCHAR(MAX)
 
----------------------------------------------------------------------------------------
-- 	set xml top table td's
-- 	设置xml顶部表格 td’s
-- 	create html table object for: #created_or_restored
-- 	为#created_or_restored创建html表格对象
set @xml_top = 
    cast(
        (select
            [database]  as 'td'
        ,   ''
        ,   [created_on]    as 'td'
        ,   ''
        ,   [restored_on]   as 'td'
        ,   ''
 
        from  #created_or_restored
        order by [database] asc
        for xml path('tr')
    ,   elements)
    as NVARCHAR(MAX)
        )
----------------------------------------------------------------------------------------
-- 	set xml mid table td's
-- 	设置xml中部表格td’s
-- 	create html table object for...
-- 	创建html表格对象为了…
----------------------------------------------------------------------------------------
-- 	format email
-- 	规定邮件格式
set @body_top =
        '<html>
        <head>
            <style>
                    h1{
                        font-family: sans-serif;
                        font-size: 110%;
                    }
                    h3{
                        font-family: sans-serif;
                        color: black;
                    }
 
                    table, td, tr, th {
                        font-family: sans-serif;
                        border: 1px solid black;
                        border-collapse: collapse;
                    }
                    th {
                        text-align: left;
                        background-color: gray;
                        color: white;
                        padding: 5px;
                    }
 
                    td {
                        padding: 5px;
                    }
            </style>
        </head>
        <body>
        <H3>' + @message_subject + '</H3>
        <h1>The following databases may require Database Mirroring:</h1>
 <h1>以下数据库可能需要数据库镜像:</h1>
        <table border = 1>
        <tr>
            <th> Database     </th>
            <th> Created_On       </th>
 
            <th> Last_Restored_On </th>
 
        </tr>'
         
set @body_top = @body_top + @xml_top + '</table>'
     
+ '</table>
        <h1>Go to the server by pasting in the following text under: Start-Run, or (Win + R)</h1>
<h1>通过粘贴下面的文本转到服务器：Start-Run, or (Win + R)</h1>
        <h1>mstsc -v:' + @server_name_basic + '</h1>'
 
+ '</body></html>'
----------------------------------------------------------------------------------------
-- 	send email.
发送邮件
if exists(select top 1 mirroring_role_desc from master.sys.database_mirroring where mirroring_role_desc = 'principal')
    begin
        EXEC msdb.dbo.sp_send_dbmail
            @profile_name       = 'SQLDatabaseMailProfile'
        ,   @recipients     = 'SQLJobAlerts@mydomain.com'
        ,   @subject        = @message_subject
        ,   @body           = @body_top
        ,   @body_format        = 'HTML';
 
    end
drop table #created_or_restored


```



[![WorksEveryTime](https://forthebadge.com/images/badges/60-percent-of-the-time-works-every-time.svg)](https://shitday.de/)

## Build-Info

| Build Quality | Build History |
|--|--|
|<table><tr><td>[![Build-Status](https://ci.appveyor.com/api/projects/status/pjxh5g91jpbh7t84?svg?style=flat-square)](#)</td></tr><tr><td>[![Coverage](https://coveralls.io/repos/github/tygerbytes/ResourceFitness/badge.svg?style=flat-square)](#)</td></tr><tr><td>[![Nuget](https://img.shields.io/nuget/v/TW.Resfit.Core.svg?style=flat-square)](#)</td></tr></table>|<table><tr><td>[![Build history](https://buildstats.info/appveyor/chart/tygerbytes/resourcefitness)](#)</td></tr></table>|

## Author

- **李聪明的数据库 Lee's Clever Data**
- **Mike的数据库宝典 Mikes Database Collection**
- **李聪明的数据库** "Lee Songming"

[![Gist](https://img.shields.io/badge/Gist-李聪明的数据库-<COLOR>.svg)](https://gist.github.com/congmingshuju)
[![Twitter](https://img.shields.io/badge/Twitter-mike的数据库宝典-<COLOR>.svg)](https://twitter.com/mikesdatawork?lang=en)
[![Wordpress](https://img.shields.io/badge/Wordpress-mike的数据库宝典-<COLOR>.svg)](https://mikesdatawork.wordpress.com/)

---
## License
[![LicenseCCSA](https://img.shields.io/badge/License-CreativeCommonsSA-<COLOR>.svg)](https://creativecommons.org/share-your-work/licensing-types-examples/)

![Lee Songming](https://raw.githubusercontent.com/LiCongMingDeShujuku/git-resources/master/1-clever-data-github.png "李聪明的数据库")

