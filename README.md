![MIKES DATA WORK GIT REPO](https://raw.githubusercontent.com/mikesdatawork/images/master/git_mikes_data_work_banner_01.png "Mikes Data Work")        

# Use SQL To Get Mirror Operating Modes Alerts When They Become High Safety Synchronous
**Post Date: October 7, 2015**        



## Contents    
- [About Process](##About-Process)  
- [SQL Logic](#SQL-Logic)  
- [Build Info](#Build-Info)  
- [Author](#Author)  
- [License](#License)       

## About-Process

<p>Here's some SQL Logic that you can throw into a nightly Job. Here's what it does: Checks to see if SQL Database Mail has been configured. If not; it will automatically configure your SQL Database Mail using generic email profile called 'SQL Database Mail'. Then it sends you an email notification when the SQL Database Mail has been configured using the name of the Database Server, and SQL Instance so you know where the email originated. All you need to do is replace the SMTP Server name with your server. Then it checks to see if you have any databases that have it's safety mode set to FULL ( ex. SET SAFETY FULL )</p>      


## SQL-Logic
```SQL
use msdb;
set nocount on
set ansi_nulls on
set quoted_identifier on
 

-- Configure SQL Database Mail if it's not already configured.
if (select top 1 name from msdb..sysmail_profile) is null
    begin

        -- Enable SQL Database Mail
        exec master..sp_configure 'show advanced options',1
        reconfigure;
        exec master..sp_configure 'database mail xps',1
        reconfigure;
 

        -- Add a profile
        execute msdb.dbo.sysmail_add_profile_sp
            @profile_name       = 'SQLDatabaseMailProfile'
        ,   @description        = 'SQLDatabaseMail';
 

        -- Add the account names you want to appear in the email message.
        execute msdb.dbo.sysmail_add_account_sp
            @account_name       = 'sqldatabasemail@MyDomain.com'
        ,   @email_address      = 'sqldatabasemail@MyDomain.com'
        ,   @mailserver_name    = 'MySMTPServerName.MyDomain.com'  
        --, @port           = ####              --optional
        --, @enable_ssl     = 1             --optional
        --, @username       ='MySQLDatabaseMailProfile'     --optional
        --, @password       ='MyPassword'           --optional
 
        -- Adding the account to the profile
        execute msdb.dbo.sysmail_add_profileaccount_sp
            @profile_name       = 'SQLDatabaseMailProfile'
        ,   @account_name       = 'sqldatabasemail@MyDomain.com'
        ,   @sequence_number    = 1;
 
        -- Give access to new database mail profile (DatabaseMailUserRole)
        execute msdb.dbo.sysmail_add_principalprofile_sp
            @profile_name       = 'SQLDatabaseMailProfile'
        ,   @principal_id       = 0
        ,   @is_default         = 1;
 

        -- Get Server info for test message
 
        declare @get_basic_server_name              varchar(255)
        declare @get_basic_server_name_and_instance_name        varchar(255)
        declare @basic_test_subject_message         varchar(255)
        declare @basic_test_body_message                varchar(max)
        set @get_basic_server_name              = (select cast(serverproperty('servername') as varchar(255)))
        set @get_basic_server_name_and_instance_name    = (select  replace(cast(serverproperty('servername') as varchar(255)), '\', '   SQL Instance: '))
        set @basic_test_subject_message         = 'Test SMTP email from SQL Server: ' + @get_basic_server_name_and_instance_name
        set @basic_test_body_message            = 'This is a test SMTP email from SQL Server:  ' + @get_basic_server_name_and_instance_name + char(10) + char(10) + 'If you see this.  It''s working perfectly :)'
 

        -- Send quick email to confirm email is properly working.
 
        EXEC msdb.dbo.sp_send_dbmail
            @profile_name   = 'SQLDatabaseMailProfile'
        ,   @recipients = 'SQLJobAlerts@MyDomain.com'
        ,   @subject        = @basic_test_subject_message
        ,   @body       = @basic_test_body_message;
 
        -- Confirm message send
        -- select * from msdb..sysmail_allitems
    end
 

-- get basic server info.
 
declare @server_name_basic      varchar(255)
declare @server_name_instance_name  varchar(255)
set @server_name_basic      = (select cast(serverproperty('servername') as varchar(255)))
set @server_name_instance_name  = (select  replace(cast(serverproperty('servername') as varchar(255)), '\', '   SQL Instance: '))
 

-- get basic server mirror role info.
 
declare
    @primary    varchar(255) = ( select @@servername )
,   @secondary  varchar(255) = ( select top 1 replace(left(mirroring_partner_name, charindex('.', mirroring_partner_name) - 1), 'TCP://', '') from master.sys.database_mirroring where mirroring_guid is not null )
,   @instance   varchar(255) = ( select top 1 mirroring_partner_instance from master.sys.database_mirroring where mirroring_guid is not null )
,   @witness    varchar(255) = ( select top 1 case mirroring_witness_name when '' then 'None configured' end from master.sys.database_mirroring where mirroring_guid is not null )
 

-- set message subject.
declare @message_subject        varchar(255)
set     @message_subject        = 'Fyi;  Synchronous Database Mirrors found on Server:  ' + @server_name_instance_name
 

-- create table to hold mirror operating modes
 
if object_id('tempdb..#mirror_operating_modes') is not null
    drop table #mirror_operating_modes
 
create table #mirror_operating_modes
    (
    [server]            varchar(255)
,   [database]          varchar(255)
,   [synchronous_mode]  varchar(255)
)
 

-- populate table #mirror_operating_modes
 
insert into #mirror_operating_modes
select
    [server]            = @@servername
,   [database]          = upper(db_name(sdm.database_id))
,   [synchronous_mode]  = case sdm.mirroring_safety_level_desc
                            when 'full' then 'SYNCHRONOUS   - HIGH SAFETY'
                            when 'off'  then 'ASYNCHRONOUS  - HIGH PERFORMANCE'
                            else 'Not Mirrored'
                          end
from
    master.sys.database_mirroring sdm
where
    db_name(sdm.database_id) not in ('tempdb')
    and sdm.mirroring_safety_level_desc = 'full'
order by
    @@servername, [database] asc
 
 

-- create conditions for html tables in top and mid sections of email.
 
declare @xml_top            NVARCHAR(MAX)
declare @xml_mid            NVARCHAR(MAX)
declare @body_top           NVARCHAR(MAX)
declare @body_mid           NVARCHAR(MAX)
 

-- set xml top table td's
-- create html table object for: #check_mirror_latency
 
set @xml_top = 
    cast(
        (select
            [server]            as 'td'
        ,   ''
        ,   [database]          as 'td'
        ,   ''
        ,   [synchronous_mode]  as 'td'
        ,   ''
 
        from  #mirror_operating_modes
        order by [server], [database] asc
        for xml path('tr')
        ,   elements)
        as NVARCHAR(MAX)
        )
 

-- set xml mid table td's
-- create html table object for: #extra_table_formatting_if_needed
/*
set @xml_mid = 
    cast(
        (select 
            [Column1]   as 'td'
        ,   ''
        ,   [Column2]   as 'td'
        ,   ''
        ,   [Column3]   as 'td'
        ,   ''
        ,   [...]   as 'td'
 
        from  #get_last_known_backups 
        order by [database], [time_of_backup] desc 
        for xml path('tr')
    ,   elements)
    as NVARCHAR(MAX)
        )
*/

-- format email
set @body_top =
        '<html>
        <head>
            <style>
                    h1{
                        font-family: sans-serif;
                        font-size: 80%;
                    }
                    h3{
                        font-family: sans-serif;
                        color: black;
                    }
 
                    table, td, tr, th {
                        font-family: sans-serif;
                        font-size: 87%;
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
 
        <h1>
        <p>
        Primary Server:  <font color="blue">' + @primary      + ' </font><br>
        Secondary Server:  <font color="blue">' + @secondary      +
                        case
                            when @secondary <> @instance then @secondary + '\' + @instance
                            else ''
                        end + '             </font><br>
                 
        Witness Server:  <font color="blue>"' + @witness      + ' </font></p>
        </h1>
          
        <h1>Note: Last Mirror Synch Info</h1>
        <table border = 1>
        <tr>
            <th> Server               </th>
            <th> Database         </th>
            <th> Synchronous Mode </th>
        </tr>'
 
set @body_top = @body_top + @xml_top +
/* '</table>
<h1>Mid Table Title Here</h1>
<table border = 1>
        <tr>
            <th> Column1 Here </th>
            <th> Column2 Here </th>
            <th> Column3 Here </th>
            <th> ...          </th>
        </tr>'        
+ @xml_mid */ 
'</table>
        <h1>Go to the server using Start-Run, or (Win + R) and type in: mstsc -v:' + @server_name_basic + '</h1>'
+ '</body></html>'
 

-- send email.
 
        if exists(select top 1 * from #mirror_operating_modes)
            begin
                exec msdb.dbo.sp_send_dbmail
                    @profile_name       = 'SQLDatabaseMailProfile'
                ,   @recipients         = 'SQLJobAlerts@MyDomain.com'
                ,   @subject            = @message_subject
                ,   @body               = @body_top
                ,   @body_format        = 'HTML';
            end
 
drop table #mirror_operating_modes
```


[![WorksEveryTime](https://forthebadge.com/images/badges/60-percent-of-the-time-works-every-time.svg)](https://shitday.de/)

## Build-Info

| Build Quality | Build History |
|--|--|
|<table><tr><td>[![Build-Status](https://ci.appveyor.com/api/projects/status/pjxh5g91jpbh7t84?svg?style=flat-square)](#)</td></tr><tr><td>[![Coverage](https://coveralls.io/repos/github/tygerbytes/ResourceFitness/badge.svg?style=flat-square)](#)</td></tr><tr><td>[![Nuget](https://img.shields.io/nuget/v/TW.Resfit.Core.svg?style=flat-square)](#)</td></tr></table>|<table><tr><td>[![Build history](https://buildstats.info/appveyor/chart/tygerbytes/resourcefitness)](#)</td></tr></table>|

## Author

[![Gist](https://img.shields.io/badge/Gist-MikesDataWork-<COLOR>.svg)](https://gist.github.com/mikesdatawork)
[![Twitter](https://img.shields.io/badge/Twitter-MikesDataWork-<COLOR>.svg)](https://twitter.com/mikesdatawork)
[![Wordpress](https://img.shields.io/badge/Wordpress-MikesDataWork-<COLOR>.svg)](https://mikesdatawork.wordpress.com/)

   
## License
[![LicenseCCSA](https://img.shields.io/badge/License-CreativeCommonsSA-<COLOR>.svg)](https://creativecommons.org/share-your-work/licensing-types-examples/)

![Mikes Data Work](https://raw.githubusercontent.com/mikesdatawork/images/master/git_mikes_data_work_banner_02.png "Mikes Data Work")

