```shell
#!/usr/bin/env bash

read -p "input java pid :" java_pid
if  [ ! -n "${java_pid}" ] ;then
    echo "you have not input a java pid!" | tee -a ${jstack_result_file}
    exit 1;
else
    echo "the java pid you input is ${java_pid}" | tee -a ${jstack_result_file}
fi

# 查询cpu最高的几个线程数
cpu_high_rate_top_num=5
# jstack 线程信息展示多少条
jstack_info_show_line=10

basedir=`cd $(dirname $0); pwd -P`
jstack_file="${basedir}/jstack_${java_pid}"
jstack_result_file="${basedir}/jstack_result_${java_pid}"

if [[ -f ${jstack_file} ]]; then
    echo "rm previous jstack file!!!" | tee -a ${jstack_result_file}
    rm -rf ${jstack_file};
fi

if [[ -f ${jstack_result_file} ]]; then
    echo "rm previous jstack result file!!!" | tee -a ${jstack_result_file}
    rm -rf ${jstack_result_file};
fi

jstack ${java_pid} > ${jstack_file}

threads_cpuinfo=`top -H -b -n1 -p ${java_pid} | awk '/PID/,0{print $1,$9}' | head -n$((${cpu_high_rate_top_num}+1))`
echo  "$threads_cpuinfo" | while read thread_cpuinfo
do
    thread_pid=`echo ${thread_cpuinfo} | cut -d " " -f 1`
    cpu_rate=`echo ${thread_cpuinfo} | cut -d " " -f 2`
    echo -e "\e[1;32;42m ********start check: { \"thread_pid\": ${thread_pid}; \"cpu_rate\": ${cpu_rate} } \e[0m" | tee -a ${jstack_result_file}
    if [[ ${thread_pid} == "PID" ]]; then
        echo "====undo"
        continue;
    fi
    thread_pid_hex_code=`printf %x ${thread_pid}`
    echo "thread pid hex code: ${thread_pid_hex_code}" | tee -a ${jstack_result_file}
    grep -A ${jstack_info_show_line} "${thread_pid_hex_code}" ${jstack_file} | tee -a ${jstack_result_file}
done
```

