# The-program-of-ARP-firewall
The ARP firewall is my first program issued in GitHub
#!/usr/bin/python3
# -*- coding: UTF-8 -*-

"""最终版"""
"""当受到主机欺骗时使用此程序进行抓数据包和广播包进行对比得出 正确的网关地址！！"""

from scapy.all import *
import time, optparse, wmi, socket


def TimeStampTime(Mytimestamp):
    MytimeTmp = time.localtime(Mytimestamp)
    MyTime = time.strftime("%Y-%m-%d %H:%M:%S", MytimeTmp)
    return MyTime


def get_itIP():
    wmi_obj = wmi.WMI()
    wmi_sql = "select IPAddress,DefaultIPGateway from Win32_NetworkAdapterConfiguration where IPEnabled=TRUE"
    wmi_out = wmi_obj.query(wmi_sql)
    return wmi_out[0].DefaultIPGateway[0]


def get_IP():
    hostname = socket.gethostname()
    MyIP = socket.gethostbyname(hostname)
    return MyIP


def CallBack(Thedata):
    try:
        print('*' * 30)
        # 用字典存储可疑Mac，并记录次数  0是本机Mac，1是网关Mac
        flag = 0
        for i in list(TheMacdict_packet.keys()):
            if Thedata[Ether].dst == i:
                flag += 1
        if flag != 0:
            TheMacdict_packet[Thedata[Ether].dst] += 1
        else:
            TheMacdict_packet[Thedata[Ether].dst] = 0

        # 保存文件
        IP_Mac_string = "北京时间:  " + TimeStampTime(Thedata.time) + \
                        "   网关IP:    " + Thedata[IP].dst + "    网关Mac地址:    " + Thedata[Ether].dst
        with open('{}.txt'.format(file_name), 'a') as file_Text:
            file_Text.write(IP_Mac_string + '\n')
        # 打印
        print("[%s] 源IP: %s---->目的IP: %s" % (TimeStampTime(Thedata.time), Thedata[IP].src, Thedata[IP].dst))
        print("[%s] 源Mac地址: %s    目的Mac地址: %s" % (TimeStampTime(Thedata.time), Thedata[Ether].src, Thedata[Ether].dst))

        ARP_send()
        print('-' * 30)
    except Exception as e:
        print(str(e))


def ARP_send():
    global TheIP, Mycount
    flag_lie = 0

    # 构造ARP包 并srp发送和接收
    try:
        (ans, unans) = srp(Ether(dst="FF:FF:FF:FF:FF:FF") / ARP(pdst=TheIP), timeout=1, verbose=0)
    except Exception as e:
        print(str(e))
    else:
        for snd, rcv in ans:
            flag_Mac = 0
            TheMac = rcv.sprintf("%Ether.src%")
            TheIP = rcv.sprintf("%ARP.psrc%")
            # 打印
            print("网关Mac:    " + TheMac + "--->" + "网关IP:     " + TheIP)
            # 保存文件
            New_string = "网关Mac:    " + TheMac + "--->" + "网关IP:     " + TheIP
            with open('{}.txt'.format(file_name), 'a') as file_Text:
                file_Text.write(New_string + '\n')

            # 将可疑Mac地址存入
            for i in TheMaclist2_packet:
                if i == TheMac:
                    flag_Mac += 1
                    break
            if flag_Mac == 0:
                TheMaclist2_packet.append(TheMac)


def judge_Mac():
    # 用于生成IP记录日志，便于后续维护和检查
    with open('{}.txt'.format(file_name), 'a') as file_Text:
        for i in TheMacdict_packet:
            file_Text.write('可疑Mac地址:    ' + i + '           出现次数   :' + str(TheMacdict_packet[i]) + '\n')
            print('可疑Mac地址:    ' + i + '           出现次数   :' + str(TheMacdict_packet[i]))
    flag_comparison = 0
    for i in TheMaclist2_packet:
        if i != list(TheMacdict_packet.keys())[0] and list(TheMacdict_packet.keys())[1]:
            print('\n' + "正确的网关是:  ", i)
            return i
        else:
            flag_comparison += 1

    if flag_comparison != 0:
        print("请确认是否存在网关欺骗，反正我是检查不到有攻击者！！")


if __name__ == '__main__':
    # ——————————参数列表————————
    TheIP = get_itIP()
    direction_limit = "host arp"
    Mycount = 1000
    file_name = 'log'
    MyIP = get_IP()
    TheMacdict_packet = {MyIP: 0}
    TheMaclist2_packet = []  # 均用于可疑存储Mac地址
    # -------------------------
    parser = optparse.OptionParser()
    # 添加 IP 参数 -i
    parser.add_option('-i', '--IP', dest='hostIP', default='{}'.format(TheIP), type='string')
    # 添加 数据包 总参数 -c
    parser.add_option('-c', '--count', dest='packetCount', default=Mycount, type='int')
    # 添加 保存文件名 参数 -o
    parser.add_option('-o', '--output', dest='fileName', default='{}.pacp'.format(file_name), type='string')

    # 解析对象和生成列表
    (options, args) = parser.parse_args()
    defFilter = direction_limit + options.hostIP
    packets = sniff(filter=defFilter, prn=CallBack, count=options.packetCount)
    TheMacdict_packet.pop(MyIP)
    # 网关的Mac地址我存这了
    Gateway_Mac = judge_Mac()  # 若发生ARP欺骗则返回正确网关地址，否则返回空
