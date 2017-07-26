---
layout: article
title:  "数据仓库之路(五)"
categories: DW
toc: true
image:
    teaser: /teaser/数据监听18511387.jpg
---

> 本文主要介绍在w公司的数据仓库之数据监控


### 前言
&emsp;&emsp;主要想记录自己在w公司以来的经历和成长，挤出些时间对自己两年以来坐下总结，以便对下一步的道路进行更好的规划。
### 文章概述
数据仓库在各种原始业务数据的抽取，转换、中间处理，加载及对外接口推送过程中，需要设置对各个环节进行监控，获取其运行的情况，同时针对不同优先级的异常信息进行告警，及早发现，以此来保证线上的仓库正常运行。
### ETL处理过程中存在的问题
   
    * Shell脚本调用第三方接口如Hive时，错误信息不能直接捕获
   
    * 部分功能的运行错误需要解析日志才能捕获，如Sqlldr中reject的行数等
   
    * 调度工具Kettle不能捕获非sh自身的错误，造成后续作业继续运行
   
    * 仓库Hadoop处理增量合并时，增量是否合并成功，是否存在当天数据等
   
## 数据监控功能概述
结合目前仓库的情况，整理梳理后，截至目前大致监控的类型可分为如下几部分:

* Hadoop文件的监控

 应用场景：主要监控HDFS上文件目录是否存在正确的分区，如大部分作业都会依赖的中间层订单表目录: /层1/层2/$table_name/dt=’$date’
{% highlight bash %}
{% raw %}
#!/bin/sh
#=====FUNCTION================================================================
#        NAME: ck_hdfs_file_exist
# DESCRIPTION: 判断指定HDFS目录是否存在文件
# PARAMETER 1: HDFS文件目录完整路径(以 \ 结尾)
# PARAMETER 2: 表名(小写)
# PARAMETER 3: 数据日期(YYYY-MM-DD)
#      RETURN: 0 --- 存在
#              1 --- 不存在
#=============================================================================
ck_hdfs_file_exist_single()
{
    if [ "$#" -ne 3 ]; then
        echo "input error: need 3 parameters"
        return 1
    fi
    local full_hdfs_path="$1"
    if [ ${full_hdfs_path##*/} ]; then
        full_hdfs_path=${full_hdfs_path}"/"
    fi
    local table_name="$2"
    local data_date="$3"

    local exists='0'
    local not_exists='1'
    local cur_cmd="ck_hdfs_file_exist"
    local log_path="/dw/log/audit/"
    local log_name="audit_`date '+%Y-%m-%d'`.log"
    local split="#"

    local log_prefix=""
    log_prefix="$log_prefix$(date '+%Y-%m-%d %H:%M:%S')"
    log_prefix="$log_prefix$split"
    log_prefix="$log_prefix$cur_cmd"
    log_prefix="$log_prefix$split"
    log_prefix="$log_prefix$data_date"
    log_prefix="$log_prefix$split"
    log_prefix="$log_prefix$table_name"
    log_prefix="$log_prefix$split"
    log_prefix="$log_prefix$full_hdfs_path"
    log_prefix="$log_prefix$split"

    local is_empty=""
    is_empty=`hadoop fs -ls "$full_hdfs_path" 2>/dev/null| wc -l 2>/dev/null`
    if [ $is_empty -eq 0 ]; then
        echo "$log_prefix$not_exists" >> "$log_path$log_name"
        #发现错误
        echo "$log_prefix$not_exists"
        ck_send_alert "ERROR" "ck_hdfs_file_exist" "$full_hdfs_path 不存在"
        return "$not_exists"
    else
        echo "$log_prefix$exists"
        echo "$log_prefix$exists" >> "$log_path$log_name"
        return "$exists"
    fi

}

{% endraw %}
{% endhighlight %}

* 文件及目录大小检查

 应用场景：仓库中有些表是采用增量合并全量的逻辑，通过判定当天分区下文件的大小是否大于昨天分区的大小，或是否在指定的域值内。
{% highlight bash %}
{% raw %}
    yes_date=`date -d "-1 day $data_date" +%Y-%m-%d`
    yes_full_hdfs_path=`echo ${full_hdfs_path/$data_date/$yes_date}`
    echo yes_full_hdfs_path:$yes_full_hdfs_path

    local dir_sum=""
    local yes_dir_sum=""
    yes_dir_sum=`hadoop fs -du $yes_full_hdfs_path \
                 | awk '{ total += $1}; END { printf "%.3f", total/1024 }'`

    dir_sum=`hadoop fs -du $full_hdfs_path \
                 | awk '{ total += $1}; END { printf "%.3f", total/1024 }'`


    yes_dir_sum=`echo $yes_dir_sum|cut -d "." -f 1`
    dir_sum=`echo $dir_sum|cut -d "." -f 1`

    echo "cut_dir_sum:"$dir_sum" cut_yes_dir_sum:"$yes_dir_sum

    threshold_file_size=`awk -v t_file_size="$yes_dir_sum" 'BEGIN{printf "%.3f", t_file_size*0.2}'`
    threshold_file_size=`echo $threshold_file_size|cut -d "." -f 1`
    echo threshold_file_size2:"$threshold_file_size"


    if [ $dir_sum -gt $threshold_file_size ]; then
        echo $log_prefix$dir_sum"KB" >> $log_path$log_name
{% endraw %}
{% endhighlight %}

* 运行日志监控

 通过分析运行日志，一方面对部分运行日志不方便实时获取到的点，需要通过分析其运行日志来捕获运行异常问题，

 应用场景：如仓库Shell中调用Hive的过程中，出现的Hsql异常，Hadoop集群资源异常或JavaOOM的问题。通过分析日志，采集运行启动时间、结束时间、处理的数据量
{% highlight bash %}
{% raw %}
#=====FUNCTION================================================================
#        NAME: ck_log_error
# DESCRIPTION: 日志中检索执行状态关键词
#     VERSION: 1.1
# PARAMETER 1: 需要监控的程序日志路径
# PARAMETER 2: 数据日期（格式：YYYY-MM-DD）
# PARAMETER 3: 作业名称
#      RETURN: 1 --- 存在错误
#              0 --- 不存在错误
#=============================================================================
ck_log_error_single()
{
    if [ "$#" -ne 3 ]; then
        echo "ck_log_error: input error: need 3 parameters"
        return 1
    fi

    local ck_path="$1"
    if [ ${ck_path##*/} ]; then
        ck_path=${ck_path}"/"
    fi
    local data_date="$2"
    local task_name="$3"

    local exists="1"
    local not_exists="0"
    local cur_cmd="ck_log_error"
    local log_path="/dw/log/audit/"
    local log_name="audit_`date '+%Y-%m-%d'`.log"
    local split="#"

    this_ini_path=`dirname $0`
    config_file="$this_ini_path/config/keywords.ini"
    echo config_file:$config_file

    local log_prefix=""
    log_prefix="$log_prefix$(date '+%Y-%m-%d %H:%M:%S')"
    log_prefix="$log_prefix$split"
    log_prefix="$log_prefix$cur_cmd"
    log_prefix="$log_prefix$split"
    log_prefix="$log_prefix$data_date"
    log_prefix="$log_prefix$split"
    log_prefix="$log_prefix$task_name"
    log_prefix="$log_prefix$split"
    log_prefix="$log_prefix$ck_path"
    log_prefix="$log_prefix$split"
    local pattern=""
    if [ -s "$config_file" ]; then
        # awk读取配置文件中设置的关键词，空行及#开头的行忽略
        pattern=$(
                     awk '
                     ($0 !~ /^[[:space:]]*#.*/) && (NF > 0) {
                         line_count += 1;
                         if (line_count == 1)
                             printf "%s", $0;
                         else
                             printf "|%s", $0;
                     }
                     END {
                         print "";
                     }' "$config_file"
                 )
    fi

    if [ ! "$pattern" ]; then
        echo "[WARN] $0: config file not exists,"\
             "using default search keywords..." >&2

        # 默认搜索关键词：正则表达式语法，已忽略大小写
        declare -a keywords
        keywords[0]='fatal\s+error'
        keywords[1]='OutOfMemoryError'
        keywords[2]='Unable\s+to\s+determine\s+Hadoop\s+version'
        keywords[3]='Java\s+Runtime\s+Environment'

        for it in "${keywords[@]}"
        do
            pattern=$pattern$it'|'
        done
        pattern=${pattern%'|'}
    fi

    local tmp_file=$(mktemp)
    #----------------------------------------------------------
    # 在指定目录下的文件中查找多个关键词，
    # 并将包含结果的行拼接成日志输出到临时文件中:
    #   find /path/to/check/ -maxdepth 1 -type f \
    #        -exec grep -n -i -E "keyword1|keyword2" {} +\
    #       | awk -v log_p="日志前缀" '{ print log_p $0; }'\
    #   >> "临时文件"
    ## find $ck_path -maxdepth 1 -type f -name "*$data_date*" \
    #----------------------------------------------------------
    find $ck_path -maxdepth 1 -type f -name "*$data_date*" \
              -exec grep -n -i -o -E "($pattern).{0,50}" {} +\
              | awk -v log_p="$log_prefix" -v log_e="$split$exists" '{ print log_p $0 log_e}'\
       >> "$tmp_file"

    if [ -s "$tmp_file" ]; then
         cat "$tmp_file" >> "$log_path$log_name"

          ck_send_alert "ERROR" "ck_log_error" "日志发生错误"
          rm -f $tmp_file
          return "$exists"
    else
         echo "$log_prefix$not_exists" >> "$log_path$log_name"
          rm -f $tmp_file
          return "$not_exists"
     fi
}

#----------------------------------------------------------
ck_log_error()
{
   local func_exec_rs=0
   source $1
   date_nano_sec=`date +%Y%m%d%H%M%S%N`
   echo "$param" | awk 'gsub(/^ *| *$/,"")'>>$1.$date_nano_sec
   while read line;
   do

    if [ 'X'"$line" != 'X' -a ${#line} -gt 1 ]; then
      echo ck_log_error_line:"$line"
      ck_log_error_single $line
      func_exec_rs=$[func_exec_rs+$?]
        #echo $func_rs
    fi

   done < $1.$date_nano_sec;

   rm -rf $1.$date_nano_sec
   return $func_exec_rs
}

{% endraw %}
{% endhighlight %}

## 元数据监控

* 数据元数据监控
    
   主要针对目前Kettle的运行日志元数据监控，通过查询Kettle的运行日志数据库，获取到某个JOB或Trans运行异常等  
* 业务元数据监控,如模型一致性等后期再考虑

## 数据质量监控
 结合目前仓库的情况，数据质量监控从如下四个方面进行监控
* 非空值检查，如交易数据中的卖家id，买家
* 数据类型检查，如日志中日期类型字段
* 业务规则检查，如商品编码长度、订单金额必须大于0等
* 一致性检查，如同层的数据效验，不同层间的数据效验等
* 完整性检查，如订单主表与订单商品表是否能全部关联上等

## 运行环境监控
 由于针对系统服务器硬件，IT运维组有独立的监控，所以本功能只考虑针对运行时空间大小进行监控，防止出现运行时数据把目录撑满，提前预防等情况，：
* HDFS目录空间(数据及临时目录)
* 共享及目录空间
* 接收日志目录空间

## 告警功能:
* 针对不同的监控问题级别，调用IT运维监控的短信报警接口或邮件接口，发送报警短信或邮件。
{% highlight bash %}
{% raw %}
#==============================================================================
#        NAME: ck_send_alert
# DESCRIPTION: 发送告警信息
# PARAMETER 1: 告警类型
# PARAMETER 2: 告警级别
# PARAMETER 3: 告警信息内容
#      RETURN: 1 --- 有错误
#              0 --- 没有错误
#==============================================================================
alert_service_name="dw_alert"
alert_mob_list="12345678123,12345678123"
localIP="172.21.150.50"
ck_send_alert()
{
  #echo '程序绝对路径为'`dirname $0`
  this_sh_abs_path=`dirname $0`
  #echo $this_sh_abs_path
  msg_00="$alert_service_name"
  msg_01="$localIP"
  msg_1="$1"   #$1:错误级别
  msg_2="$2"   #$2:ck检查名
  msg_3="$3"   #$3:content
  repeat_time=5
  local msg_log_name="test_audit(`date '+%Y-%m-%d'`).log"
  send_msg="$msg_00 $msg_01 $msg_1 $msg_2 $msg_3"
  #echo "$(date '+%Y-%m-%d %H:%M:%S') $msg" >> $this_sh_abs_path/.
  echo msg:"$send_msg"
#调取运维的短信监控接口
  wget  --output-document=/dev/null "http://hostname/message.php?phone=$alert_mob_list&msg=$send_msg"
}

{% endraw %}
{% endhighlight %}

## 功能设计
* HDFS文件的监控
{% highlight bash %}
{% raw %}
在仓库的每个Shell脚本最后，调用【文件及目录存在性检查】函数，用于检查Shell处理的HDFS目录存在，日志ID ck_hdfs_file_exist
  方法名：ck_hdfs_file_exist_single
  输入参数1:  HDFS文件目录完整路径(以 \ 结尾)
  输入参数2:  表名(小写)
  输入参数3:  数据日期(YYYY-MM-DD)
中间过程：
  a) 判定目录文件hadoop fs -ls是否存在，记录日志到目录
  
  b) 如不存在，则调用告警接口
  输出参数: 1不存在 0存在
  输出日志:
   日志文件 /log/audit/audit_YYYY-MM-DD.log
   日志格式 系统时间#ck_hdfs_file_exist#ETL日期#表名#完整目录#检查结果
  方法名：ck_hdfs_file_exist
  输入参数1:  批处理时的配置文件
   ./config/ods_ck_hdfs_file_exist.ini
输出参数:   数据执行标志，为0是代表全部成功，大于0代表有错误发生
中间过程：a) 循环读取配置文件中的参数，调用ck_hdfs_file_exist_single
   b) 将每次循环执行的返回值累加，最后返回累计的结果值
{% endraw %}
{% endhighlight %}
* 文件及目录大小检查
{% highlight bash %}
{% raw %}
   在仓库的每个Shell脚本最后，调用【文件及目录大小检查】函数，用于检查Shell处理的HDFS目录大小，
   方法名：ck_hdfs_file_size_single()
   输入参数1:  文件目录完整路径
   输入参数2:  表名(小写)
   输入参数3:  数据日期(YYYY-MM-DD)
   输出参数 :  1效验不通过 0效验通过
   中间过程：  调用hadoop fs –du，根据传入的参数统计目录大小(以KB为单位)，同时计算前一天的目录大小，如果当天的文件目录大小值小于前一天的20%，则认为检查不通过同时触发报警，记录日志。

   输出日志:
   日志文件 /dw/log/audit/audit_YYYY-MM-DD.log
   日志格式 系统时间#ck_hdfs_file_size#ETL日期#表名#完整目录#目录大小

   

   方法名：ck_hdfs_file_size()
   输入参数1:  批处理时的配置文件
   ./config/ods_ck_hdfs_file_size.ini
   输出参数:   数据执行标志，为0是全部效验通过，大于0代表没效验通过
   中间过程：a) 循环读取配置文件中的参数，调用ck_hdfs_file_size_single
            b) 将每次循环执行的返回值累加，最后返回累计的结果值

{% endraw %}
{% endhighlight %}

* 数据质量监控
  用于检查数据整体质量情况
{% highlight bash %}
{% raw %}
  方法名：ck_data_dq_single()
  输入参数1: 质量检查ID (具体每个检查点的编码，不能为空) 
  输入参数2: 日期
  输入参数3: 
  输出参数: 1执行成功 0执行失败
  中间过程: a) 根据检查id调用hsql文件夹下同名的检查Hsql，
     通过hive -S –e执行将执行结果记录下运行结果。
   b) 如有不一致则不通过，调用告警接口


  输出日志:
     日志文件 /dw/log/audit/audit_YYYY-MM-DD.log
     日志格式 系统时间#ck_data_dq#ETL日期#质量检查ID#完整目录# 



    方法名：ck_data_dq ()
    输入参数1:  批处理时的配置文件
                ./config/ods_ck_data_dq.ini
                ./config/mds_ck_data_dq.ini
                
     输出参数:   数据执行标志，为0是全部效验通过，大于0代表没效验通过
     中间过程：a) 循环读取配置文件中的参数，调用ck_data_dq_single
              b) 将每次循环执行的返回值累加，最后返回累计的结果值

{% endraw %}
{% endhighlight %}
* 告警
{% highlight bash %}
{% raw %}
  用于调用短信或邮件接口
          方法名：ck_send_alert
          输入参数1: 告警级别（ERROR，DEBUG，INFO等）
          输入参数2: 告警(1紧急 2一般 3忽略)
          输入参数3: 告警信息内容
          输出参数: 1执行成功 0执行失败
          中间过程: 调用IT运维短信接口或邮件接口，发送告警。
{% endraw %}
{% endhighlight %}