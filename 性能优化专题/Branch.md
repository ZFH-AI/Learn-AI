# 1、分支预测实现

```python
#  功能说明
#  1、根据提供的文件目录或单个文件按照既定的IF类型分析每个FILE中的IF占比情况
#  2、根据提供的文件目录或单个文件获得该FILE的代码覆盖率网页，分析循环执行次数和循环内的IF执行次数

import csv
import os
import re
import requests
import json
from bs4 import BeautifulSoup
import urllib3
import pandas as pd

urllib3.disable_warnings()

# 定义爬虫使用的全局变量
PRIVATE_TOKEN = ''
USER_AGENT = ''
COOKIES = ''
HOST = ''
headers = {'Token': PRIVATE_TOKEN,
           'Cookie': COOKIES,
           'Host': HOST,
           'User-Agent': USER_AGENT}

# 定义代码覆盖率网页API使用的全局变量
URL = ''
REPO_PATH = ''
SRC_PART1 = ''


# 同级目录下的配置文件读入
def WriteConfigFromLocalFile():
    ret = 1
    global PRIVATE_TOKEN, USER_AGENT, URL, REPO_PATH, SRC_PART1, COOKIES, HOST, headers
    with open(r'config.txt', 'r', encoding='utf-8-sig') as f:
        try:
            data_dict = {}
            lines = f.read().splitlines()
            for line in lines:
                item = line.strip().split(':')
                key = item[0]
                value = item[1]
                if item[0] == 'URL':
                    value += ':' + item[2]
                data_dict[key] = value
            if len(data_dict) == 0:
                print(r" 配置文件D:\config.txt中无配置内容，请填写 ！！")
                return

            PRIVATE_TOKEN = data_dict['PRIVATE_TOKEN']
            COOKIES = data_dict['COOKIES']
            HOST = data_dict['HOST']
            USER_AGENT = data_dict['USER_AGENT']
            URL = data_dict['URL']
            REPO_PATH = data_dict['REPO_PATH']
            SRC_PART1 = data_dict['SRC_PART1']
            headers = {'Token': PRIVATE_TOKEN,
                       'Cookie': COOKIES,
                       'Host': HOST,
                       'User-Agent': USER_AGENT}
        except IOError:
            print("错误警告: 没有找到文件或读取文件失败")
            ret = 0
        finally:
            f.close()
    return ret


# 根据文件名称删除该文件
def RemoveFileByName(fileName):
    if len(fileName) == 0:
        print(f"输入的文件名称长度为零，请检查！！")
        return
    if os.path.exists(fileName):
        if os.access(fileName, os.W_OK):
            os.remove(fileName)
        else:
            print(f"警告: 文件 {fileName} 当前被其他程序打开。请关闭该文件后再尝试运行程序")
    else:
        return


# 将文件路径转成浏览器识别的路径
def filePathTranslation(filePath):
    originPath = filePath.split("\\")
    filtered_str = originPath[3:]
    res = "/".join(filtered_str)
    return res


# 将代码覆盖率网页中的IF行数、执行次数、IF内容等信息写入到Excel中
def WriteAllCodeIfInfoToExcel(resultList, failedList):
    fileName = r'D:/IF_Information_Details.xlsx'
    RemoveFileByName(fileName)
    writer = pd.ExcelWriter(fileName, engine='xlsxwriter')
    IfTypeRows = []
    Title = ['文件名称', 'IF所在行号', 'IF执行次数', 'IF语句体执行次数', 'IF判断所属类型', 'IF判断所属子类型', 'IF判断表达式',
             '解决方案', '覆盖率链接']
    for item in resultList:
        last_name = item[0].split("\\")[-1]
        ifInfo = item[1]
        url = item[2]
        ifInfoLen = len(ifInfo)
        for i in range(ifInfoLen):
            IfTypeRows.append([last_name, ifInfo[i][0], ifInfo[i][1], ifInfo[i][2], '', '', ifInfo[i][3], '', url])
    df = pd.DataFrame(IfTypeRows, columns=Title)
    df.to_excel(writer, sheet_name="每个FILE中IF关键字的总明细", index=False)

    Title1 = ['该FILE无覆盖网页在[5g.rnd.huawei.com]中']
    df1 = pd.DataFrame(failedList, columns=Title1)
    df1.to_excel(writer, sheet_name="文件无对应的代码覆盖率-汇总分析", index=False)

    writer.save()


#  文件目录处理类
class FileDirectoryProcessing:
    def __init__(self, filePath):
        self.filePath = filePath

    def ProcessFileDir(self, Path, fileList):
        fileDir = os.listdir(Path)
        for fileName in fileDir:
            filePath = os.path.join(Path, fileName)
            # 判断是否为文件
            if os.path.isfile(filePath):
                fileList.append(filePath)
            # 判断是否为目录
            elif os.path.isdir(filePath):
                self.ProcessFileDir(filePath, fileList)

    def GetFileRootPathList(self):
        fileList = []
        if os.path.isfile(self.filePath):
            fileList.append(self.filePath)
        elif os.path.isdir(self.filePath):
            self.ProcessFileDir(self.filePath, fileList)
        else:
            print('Error: Invalid Path!')
        return fileList


# 读取API获得文件对应的HTML的URL
def GetUrlByFilePath(filePath, failedList):
    params = {
        "repo_path": REPO_PATH,
        "src_path": SRC_PART1 + filePath,
    }
    response = requests.get(url=URL, params=params, headers=headers, verify=False)
    if response.status_code != 200:
        failedList.append(f"该文件获取代码覆盖率链接失败!! {filePath} \n")
        return None
    url = response.json()['itst_url']
    return url


# 根据URL获得该HTML中的内容
def getHtmlTxtByUrl(url, header):
    txtContent = ""
    res = requests.get(url=url, headers=header, verify=False)
    if res.status_code != 200:
        return txtContent
    # 按照标签提取内容
    soup = BeautifulSoup(res.content, 'html.parser')
    span_tags = soup.find_all('pre', class_=['source'])

    for item in span_tags:
        txtContent += item.text
    return txtContent


# 根据URL获得HTMl的内容
def GetHtmlTxtByUrl(url, failedList):
    txtContent = getHtmlTxtByUrl(url, headers)
    if len(txtContent) == 0:
        failedList.append(f"无法获取该HTMl的内容 {url} \n")
        return None
    return txtContent


#  获得待码覆盖率网页中的行号
def GetLineNum(lineStr):
    if lineStr.isspace() or len(lineStr) == 0:
        return 0
    lineNum = re.findall(r'\d+', lineStr)
    return lineNum[0]


# 输入列表返回if(){}中的 { 的所在位置下标
def GetIfLeftBraceIndex(if_item):
    for index, item in enumerate(if_item):
        if "{" in item:
            return index
    return -1


#  获得待码覆盖率网页中的执行次数
def GetExcTime(timeStr):
    if timeStr.isspace() or len(timeStr) == 0:
        return 0
    excTime = re.sub(r'\s+', '', timeStr)  # 将多个连续的空格替换为一个空格
    excTime = re.findall(r'\d+', excTime)  # 提取出数字
    return excTime[0]


#  获得待码覆盖率网页中的执行次数
def GetIfJudgeItemExcTime(if_Txt):
    ifList = if_Txt.split('\n')
    ifListLen = len(ifList)
    rowIndex = 0
    while rowIndex < ifListLen:
        item = ifList[rowIndex]
        itemList = item.split(':')
        if GetIfLeftBraceIndex(itemList) != -1:
            # 单独处理IF语句体的行号
            rowIndex += 1
            while rowIndex < ifListLen:
                subItemList = ifList[rowIndex].split(":")
                subItemListLen = len(subItemList)
                if subItemListLen < 2:
                    rowIndex += 1
                    continue
                excTime = subItemList[1]
                excTime = re.sub(r'\s+', '', excTime)
                excTime = re.findall(r'\d+', excTime)
                if len(excTime) == 0:
                    rowIndex += 1
                else:
                    return excTime[0]
        rowIndex += 1
    return -1


#  获得IF()内容
def GetIfJudgeContent(i, if_item2len, if_item2):
    ifTxtContent = ''
    while i < if_item2len:
        strTxt = if_item2[i]
        # 寻找换行符 \n 如存在则保留\n之前的内容
        n_index = strTxt.find('\n')  # 找到\n的位置
        if n_index != -1:
            strTxt = strTxt[:n_index] + ' '
        # 将多个连续的空格替换为空字符
        strTxt = re.sub(r'\s+', ' ', strTxt)
        # 如果strTxt是纯数字则替换为空字符
        if re.fullmatch(r'\d+', strTxt):
            strTxt = re.sub(r"\b\d+(?!\))\b", ' ', strTxt)  # 如果是xx)这样的则不替换，含义是判断语句中有数字
        # 将大括号{后所有的字符替换为空字符
        strTxt = re.sub(r'{\s*.*', '', strTxt)
        if len(strTxt) == 0 or strTxt.isspace():
            i += 1
            continue
        ifTxtContent += strTxt
        i += 1

    ifTxtContent = re.sub(r'\s+', ' ', ifTxtContent)  # 将多个连续的空格替换为空字符
    return ifTxtContent


# 获取代码覆盖率网页上的IF的行号：执行次数：IF判断语句
def GetFileALLIfInfoWithUrl(txtContent):
    ifInfoList = []  # [IF行号，IF执行次数，IF语句体执行次数，IF语句判断内容]
    # ifReg = r'\d+([\s]*.*?)\bif\s*\(([^{}]+)\)\s*\{'
    ifReg = r'\d+([\s]*.*?)\bif\s*\(([^{}]+)\)\s*\{(([\s\S]*?)(?:[^{}]*(?:\{([\s\S]*?)[^{}]*\}))*[^{}]*?)\}'
    if_matches = re.finditer(ifReg, txtContent)
    if not if_matches:
        return ifInfoList

    for im in if_matches:
        if_Txt = im.group()
        # print(f"if_TXT\n {if_Txt}")
        if_item = if_Txt.split(':')
        # print(f"if_item\n {if_item}")
        if_itemLen = len(if_item)
        if_LineNum = GetLineNum(if_item[0])
        if_ExcTime = GetExcTime(if_item[1])
        # if () { }中的 { 的下标所在的位置
        ifLeftBrace = GetIfLeftBraceIndex(if_item)
        startIndex = 2  # 列表开始处理的位置
        endIndex = min(ifLeftBrace, if_itemLen) + 1  # 列表结束处理的位置
        # 获得IF判断体的内容
        ifTxtContent = GetIfJudgeContent(startIndex, endIndex, if_item)
        # 获得IF语句体的执行次数
        ifJudgeItemExcTime = GetIfJudgeItemExcTime(if_Txt)
        # print(f"ifLeftBrace= {ifLeftBrace}, startIndex = {startIndex}, endIndex = {endIndex}"
        #       f"IF Line = {if_LineNum}, Time = {if_ExcTime}, IfExpression={ifTxtContent},"
        #       f" ifJudgeItemExcTime = {ifJudgeItemExcTime}")
        ifInfoList.append([if_LineNum, int(if_ExcTime), int(ifJudgeItemExcTime), ifTxtContent])

    return ifInfoList


def GetIfCodeCoverageDetailFormOnLine():
    print(f">> 请输入要分析的文件路径[单个文件或文件目录均可]")
    file_path = str(input())
    print(f">> 你输入的内容是：" + file_path + " 对输入的内容针对 [IF分支] 的分析详情如下:")
    #  文件目录处理：获得FILE路径列表和FILE种类列表
    fileDirProcess = FileDirectoryProcessing(file_path)
    filePathLists = fileDirProcess.GetFileRootPathList()
    if len(filePathLists) == 0:
        print(f"你输入的{file_path}下没有合法的文件路径，请检查！！！ ")
    print(f"待处理的文件总数: {len(filePathLists)}")
    # 对resultList的解析:  filePathList中共有文件K个，resultList =[item*K],其中每个item组成如下
    # item：[文件名称，[IF的行数，IF的执行次数，IF判断语句内容]*N]，URL链接]
    resultList = []
    failedList = []  # 获取覆盖率失败的文件记录
    for subFile in filePathLists:
        filePath = filePathTranslation(subFile)
        url = GetUrlByFilePath(filePath, failedList)
        if url is None:
            continue
        txtContent = GetHtmlTxtByUrl(url, failedList)
        if txtContent is None:
            continue
        item = GetFileALLIfInfoWithUrl(txtContent)
        if len(item) != 0:
            resultList.append([subFile, item, url])

    # 将FILE的IF信息保存到Excel中
    WriteAllCodeIfInfoToExcel(resultList, failedList)


def GetLocalFile():
    print(f">> 请输入要分析的文件名称")
    filePath = str(input())
    print(f">> 你输入的内容是：" + filePath + " 对输入的内容针对 [IF分支] 的分析详情如下:")
    listUrl = []
    with open(filePath, "r", encoding='utf-8-sig') as f:
        try:
            listUrl = f.read().splitlines()
        except IOError:
            print("错误警告: 没有找到文件或读取文件失败")
        finally:
            f.close()
    return listUrl


def GetUrlFormLocalFile():
    listUrl = GetLocalFile()
    if len(listUrl) == 0:
        print(f"输入的文件中无内容，请核对和在输入！！")
        return
    print(f"待处理的文件总数: {len(listUrl)}")

    # 对resultList的解析:  filePathList中共有文件K个，resultList =[item*K],其中每个item组成如下
    # item：[文件名称，[IF的行数，IF的执行次数，IF判断语句内容]*N]，URL链接]
    resultList = []
    failedList = []  # 获取覆盖率失败的文件记录
    # listUrl = [
    #     'https://webide-x86-sz17.starling.huawei.com:31731/ulsch/framework/usrmng/usrlistagg/src/ulsch_usrmng_usrgrp.c.gcov.html']
    for url in listUrl:
        txtContent = GetHtmlTxtByUrl(url, failedList)
        if txtContent is None:
            continue
        item = GetFileALLIfInfoWithUrl(txtContent)
        if len(item) != 0:
            last_name = url.split("/")[-1]
            last_name = last_name.rsplit('.', 2)[0]
            resultList.append([last_name, item, url])

    # 将FILE的IF信息保存到Excel中
    WriteAllCodeIfInfoToExcel(resultList, failedList)


# 静态代码分析-IF分析的主函数
if __name__ == '__main__':
    print('+-+-+-+-+-+-+-+-+-+-+-+-+-+-+重要提示+-+-+-+-+-+-+-+-+-+-+-+-++-+-+-+-+-+')
    print(f"处理的结果保存位置 D:IF_Information_Details.xlsx")
    print(f"同级目录下有config.txt，视情况修改")
    # 读取配置配置
    if WriteConfigFromLocalFile() == 0:
        print(f"在读取同级目录下的config.txt时出现错误，程序终止运行！！")
    print('+-+-+-+-+-+-+-+-+-+-+-+-+-请选择处理模式+-+-+-+-+-+-+-+-+-+-+-++-+-+-+-+-+')
    print(f"    1: 输入文件名称或者目录，读取 5g.rnd.huawei.com 中的代码覆盖率信息")
    print(f"    2: 输入自定义的TEST的代码覆盖率路径文件信息")
    print('+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-++-+-+-+-+-+')
    print(f">> 请输入选择： 输入 1 或 2 ")
    choosePattern = int(input())
    print('+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-++-+-+-+-+-+')
    if choosePattern == 1:
        GetIfCodeCoverageDetailFormOnLine()
    elif choosePattern == 2:
        GetUrlFormLocalFile()
    else:
        print(f"没有第三个选项 :)")

    print("\n >> 恭喜哦，程序已成功执行完毕 :)")
    print("\n >>  按下任意键关闭窗口...")
    input()

```


# 2、分支分析实现-2
```python
#  功能说明
#  1、根据提供的文件目录或单个文件按照既定的IF类型分析每个FILE中的IF占比情况
#  2、根据提供的文件目录或单个文件获得该FILE的代码覆盖率网页，分析循环执行次数和循环内的IF执行次数
import csv
import os
import re
import requests
import json
from bs4 import BeautifulSoup
import urllib3
import pandas as pd

urllib3.disable_warnings()

#  定义常量
IF_TYPE_NUM = 8  # IF判断类型的种类
OUTPUT_LOOP_TP = 10  # 输出TOP10的LOOP-IF-NUM

#  定义爬虫使用的固定变量
PRIVATE_TOKEN = 'effc6ea7be43cf7df24ba73e33c4c4b3'
USER_AGENT = 'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/119.0.0.0 ' \
             'Safari/537.36'
URL = r"http://5g.rnd.huawei.com/api/cov/query/?"
REPO_PATH = r"5G_BTS/WN_5G_BTS_L2L3_24B"
SRC_PART1 = r"/5g_build/5g_Main/"
COOKIES = r'idss_cid=7d00a672-b6a6-48c2-b15c-aac5c0b5d189; cookie_version=24B; uid=1701240633-1670411157; _frid=c148a291b46d4aeab1602dd4fcf97daa; HW3MS_think_language=zh-cn; sensorsdata2015jssdkcross=%7B%22distinct_id%22%3A%223f7c4712f5d2467274e2ee645e198f3b%22%2C%22%24device_id%22%3A%2218c1a20fd9cb46-033e57bfaecd47-26031051-2073600-18c1a20fd9d27f%22%2C%22props%22%3A%7B%22%24latest_traffic_source_type%22%3A%22%E7%9B%B4%E6%8E%A5%E6%B5%81%E9%87%8F%22%2C%22%24latest_referrer%22%3A%22%22%2C%22%24latest_search_keyword%22%3A%22%E6%9C%AA%E5%8F%96%E5%88%B0%E5%80%BC_%E7%9B%B4%E6%8E%A5%E6%89%93%E5%BC%80%22%7D%2C%22first_id%22%3A%2218c1a20fd9cb46-033e57bfaecd47-26031051-2073600-18c1a20fd9d27f%22%2C%22identities%22%3A%22eyIkaWRlbnRpdHlfY29va2llX2lkIjoiMThjMWVlZmMxMWYxMmQ1LTBmNDIyZDVlZTI5MjgxOC0yNjAzMTA1MS0yMDczNjAwLTE4YzFlZWZjMTIwMTEyOSIsIiRpZGVudGl0eV9sb2dpbl9pZCI6IjNmN2M0NzEyZjVkMjQ2NzI3NGUyZWU2NDVlMTk4ZjNiIn0%3D%22%2C%22history_login_id%22%3A%7B%22name%22%3A%22%24identity_login_id%22%2C%22value%22%3A%223f7c4712f5d2467274e2ee645e198f3b%22%7D%7D; HW3MS_ResourceLanguage=czowOiIiOw%3D%3D; prod_J_SESSION_ID=e11aab811d481aa4d2501f1ea8ce1ad864ae4563524400ed; lang=zh; env_token=pro; prod_agencyID=z00465944; u_depart=%E5%8D%8E%E4%B8%BA%E6%8A%80%E6%9C%AF%2FICT%E4%BA%A7%E5%93%81%E4%B8%8E%E8%A7%A3%E5%86%B3%E6%96%B9%E6%A1%88%2F%E6%97%A0%E7%BA%BF%E7%BD%91%E7%BB%9C%E4%BA%A7%E5%93%81%E7%BA%BF%2F%E6%97%A0%E7%BA%BF%E7%BD%91%E7%BB%9C%E7%A0%94%E5%8F%91%E7%AE%A1%E7%90%86%E9%83%A8%2F%E6%97%A0%E7%BA%BF%E7%BD%91%E7%BB%9C%E4%BA%A7%E5%93%81%E5%B7%A5%E7%A8%8B%E4%B8%8EIT%E8%A3%85%E5%A4%87%E9%83%A8; prod_cftk=MFUV-1YKL-8SBC-YMRB-FGU2-2OFQ-ULBC-YRGW#KLL7-VMUZ-I4FD-5NSH-50EK-EPP0-73WP-9W3W; login_uid=4C-4C-85-D2-FA-C2-43-05-0F-C6-70-02-04-B3-7E-48; login_sid=37-1F-B1-33-C7-A0-77-60-CE-5B-5E-B8-69-B2-A6-CC-E5-33-00-27-FB-96-83-27-34-5F-5C-E4-3F-47-11-C4-BD-B0-89-4D-C4-54-80-37; login_logFlag=in; suid=4C-4C-85-D2-FA-C2-43-05-0F-C6-70-02-04-B3-7E-48; hwssot3=30624553172602; hwsso_login=V004lRYzCnFKGnUHwDGxOddPPVne5BfD8DLgVYnhyBczQZg8we9yVLxGthkgqiD3GK9N1L2g2xzJQ_avfGWaoAhQkGHSacFIoPJb87P2sr9jSzamtrmo8Sqs4c3ng8RHEQZJD_aeTq3uOo99C6zEMwjwxRi8wGz9qqPyq2S5bJhTFlOxYKc2057KJeM5zRu7eoGhu0J7YntxYDPQRzt_apR1xk8lHwKPG_b7bUW3GNX6A7IffEQuxc6Y0T3UMU2PRRBUfO_aqnF8ptj_bmtbmds98pZQEMgsurHt3jVrLCBcLTD5N9brXgy7UP5Hu4IWbJ89baAEDiN2QKYKn5cmTchxKjYQ6WSw_c_c; hwssot=15-A9-34-61-F2-26-66-15-66-AC-AE-94-11-D4-37-72; sessionid=jh7x25f1fgowpn8bwlix2j2sy3zr2wgu; ztsg_ruuid=1273d586d214629d-9d72-41e2-8734-7892303282e2'
HOST = r'5g.rnd.huawei.com'
headers = {'Token': PRIVATE_TOKEN,
           'Cookie': COOKIES,
           'Host': HOST,
           'User-Agent': USER_AGENT}


# 根据文件名称删除该文件
def RemoveFileByName(fileName):
    if len(fileName) == 0:
        print(f"输入的文件名称长度为零，请检查！！")
        return
    if os.path.exists(fileName):
        os.remove(fileName)


# 将文件路径转成浏览器识别的路径
def filePathTranslation(filePath):
    originPath = filePath.split("\\")
    filtered_str = originPath[3:]
    res = "/".join(filtered_str)
    return res


# 获得占比
def GetRatio(Molecules, Denominator):
    if Denominator != 0:
        ration = round(float(Molecules) / Denominator, 3)
    else:
        ration = 0
    return ration


# 将每个FILE中循环内的IF种类和IF执行次数累加
def CalculateIfTypeAndExc(infoList, ifType):
    for i in range(IF_TYPE_NUM + 1):
        ifType[i] += infoList[i]


# 将全部IF的静态分析结果写入Excel文件
def WriteStaticIfInfoToExcel(TotalFileIfTypeListAndExcTimes, fileTypes, fileTypeOnIfNumList, allFileIfTypeList,
                             otherTraceList, filePath):
    # 写入Excel
    last_segment = filePath.split("\\")[-1]
    fileName = r'D:/' + last_segment + '_Total_IfStatic_Analysis.xlsx'
    RemoveFileByName(fileName)
    writer = pd.ExcelWriter(fileName, engine='xlsxwriter')
    # 输出FILE-TYPE信息
    FileTypeRows = []
    Title = ['文件类型', '文件个数', '类型占比']
    fileInfoList = [".C文件:   ", ".H文件:   ", ".TXT文件： ", "OTHER文件:"]
    fileInfoLen = len(fileInfoList)
    totalFileNum = int(fileTypes[4])
    for i in range(fileInfoLen):
        ration = GetRatio(fileTypes[i], totalFileNum)
        FileTypeRows.append([fileInfoList[i], fileTypes[i], ration * 100])
    FileTypeRows.append(['总计', totalFileNum, 1])
    df = pd.DataFrame(FileTypeRows, columns=Title)
    df.to_excel(writer, sheet_name="文件类型分析", index=False)

    # 输出文件类型中的IF关键字个数
    ifRows = []
    Title1 = ['文件类型', 'IF关键字个数', '类型占比']
    fileIFNumInfo = [".C文件", ".H文件", "OTHER文件"]
    fileIfNumLen = len(fileIFNumInfo)
    totalIfNum = int(fileTypeOnIfNumList[3])
    for j in range(fileIfNumLen):
        ration = GetRatio(fileTypeOnIfNumList[j], totalIfNum)
        ifRows.append([fileIFNumInfo[j], fileTypeOnIfNumList[j], ration])
    ifRows.append(['总计', totalIfNum, 1])
    df1 = pd.DataFrame(ifRows, columns=Title1)
    df1.to_excel(writer, sheet_name="不同文件类型中IF分析", index=False)

    # 打印IF关键字划分的不同类型的具体信息
    ifDetailRows = []
    Title2 = ['IF类别检查名称', '检查出的个数', '类型占比', '备注说明']
    IfTypeInfo = ["返回值检查", "指针判空检查", "跟踪开关检查", "强值类型判断", "大颗粒特性判断", "关系运算判断", "函数式判断语句", "OTHER类型"]
    InfoBubble = ['返回值检查：形如IF(ret != L2_OK)',
                  '指针判空检查：形如IF(IS_PTR_INVALID())',
                  '跟踪开关检查：形如IF(IS_TRACE_ON()) 或者 IF(**Tr**ON)',
                  '强值类型判断：形如IF(IS_USRID_INVALID())格式',
                  '大颗粒特性判断：形如IF(F_DELETE(XX)) OR IF(F_NON_DELETE(XX))',
                  '关系运算判断：形如IF([对象][->]属性 >|>=|<|<=|==|!= [对象][->]属性)格式',
                  '函数式判断语句：形如IF(func())格式',
                  'OTHER类型：形如IF(xx-flag)格式'
                  ]
    ifTypeLen = len(allFileIfTypeList)
    for k in range(ifTypeLen):
        ration = GetRatio(allFileIfTypeList[k], totalIfNum)
        ifDetailRows.append([IfTypeInfo[k], allFileIfTypeList[k], ration, InfoBubble[k]])
    ifDetailRows.append(['总计', totalIfNum, 1, ''])
    df2 = pd.DataFrame(ifDetailRows, columns=Title2)
    df2.to_excel(writer, sheet_name="全部IF判断类型-汇总分析", index=False)

    # 将循环内的IF分类和执行汇总情况输出到Excel中
    TotalIfType = [0] * (IF_TYPE_NUM + 1)
    TotalIfExcTime = [0] * (IF_TYPE_NUM + 1)
    for item in TotalFileIfTypeListAndExcTimes:
        # print(item[0], item[1], item[2])
        CalculateIfTypeAndExc(item[1], TotalIfType)
        CalculateIfTypeAndExc(item[2], TotalIfExcTime)
    Title3 = ['循环内IF类别检查名称', '循环内IF种类检查出的个数', '类型占比', '循环内IF种类执行的次数', '次数占比', '备注说明']
    ifInfoRows = []
    for n in range(IF_TYPE_NUM):
        ration1 = GetRatio(TotalIfType[n], TotalIfType[IF_TYPE_NUM])
        ration2 = GetRatio(TotalIfExcTime[n], TotalIfExcTime[IF_TYPE_NUM])
        ifInfoRows.append([IfTypeInfo[n], TotalIfType[n], ration1, TotalIfExcTime[n], ration2, InfoBubble[n]])
    ifInfoRows.append(['总计', TotalIfType[IF_TYPE_NUM], 1, TotalIfExcTime[IF_TYPE_NUM], 1, ''])
    df3 = pd.DataFrame(ifInfoRows, columns=Title3)
    df3.to_excel(writer, sheet_name="全部循环内IF判断类型-汇总分析", index=False)

    # 将Trance 按类别写入
    # file.writelines(f'+-+-+--跟踪开关的判断补充说明-+-当前判断宏为：TRACE_PROC 和 ORIGIN_TRACE_PROC+-+\n')
    # file.writelines(f">> IF(Trance)的个数 {allFileIfTypeList[2]}, 宏(Trance)的个数 {otherTranceNum}\n")
    # file.writelines(f"++. IF(Trance) + 宏(Trance) / 宏(Trance) + TOTAL-IF = {trcNumRation:.3%}\n")
    # file.writelines(f'+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-++-+-+-+-+-+\n')

    writer.save()


# 将全部文件中每个FILE的IF静态分析结果写入Excel中
def WriteAllFileIfTypeToExcel(eachFileIfType):
    if len(eachFileIfType) == 0:
        return
    fileName = r'D:/IfType_EachFile_Analysis.xlsx'
    RemoveFileByName(fileName)
    writer = pd.ExcelWriter(fileName, engine='xlsxwriter')
    IfTypeInfo = ["返回值检查", "指针判空检查", "跟踪开关检查", "强值类型判断", "大颗粒特性判断", "关系运算判断", "函数式判断语句",
                  "OTHER类型"]
    Title = ['文件名称', 'IF类别检查名称', 'IF检查出的个数', 'IF检查个数占比']
    ifTypeRows = []
    for item in eachFileIfType:
        name = item[0].split("\\")[-1]
        ifCount = item[1]
        ifList = item[2]
        for i in range(IF_TYPE_NUM):
            ration = GetRatio(ifList[i], ifCount)
            ifTypeRows.append([name, IfTypeInfo[i], ifList[i], ration])
        ifTypeRows.append(['小计', 'IF-TYPE', ifCount, 1])
    df = pd.DataFrame(ifTypeRows, columns=Title)
    df.to_excel(writer, sheet_name="当期目录下所有FILE内全部IF判断种类-汇总分析", index=False)
    writer.save()


# 输出不同FILE下的IF判断数目
# 参数：文件类型列表、不同文件类型中IF关键字总计列表、IF-TYPE列表、文件目录
def PrintIfStatistInfo(fileTypes, fileTypeOnIfNumList, allFileIfTypeList, otherTraceList, filePath):
    print(f'+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-++-+-+-+-+-+')
    print(f'>> 在 D:\\ 盘下可能会输出如下文件分别是：')
    print(f"  XXX_Total_IfStatic_Analysis.xlsx:\t当前目录下所有FILE的全部IF和全部循环内IF的汇总分析")
    print(f"  IfInLoopType_EachFile_CoverageAnalysis.txt：\t当前目录下每个FILE中循环和循环内IF的覆盖率信息")
    print(f"  IfInLoopType_EachFile_Analysis.xlsx:\t当前目录下每个FILE中循环内IF的判断种类和覆盖率信息")
    print(f"  IfType_EachFile_Analysis.xlsx：\t当前目录下每个FILE中IF判断种类信息")

    # 打印IF关键字划分的不同类型的具体信息
    totalIfNum = int(fileTypeOnIfNumList[3])
    IfTypeInfo = ["返回值检查", "指针判空检查", "跟踪开关检查", "强值类型判断", "大颗粒特性判断", "关系运算判断",
                  "函数式判断语句", "OTHER类型判断"]
    print(f'+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-++-+-+-+-+-+')
    print(f">>  IF关键字总数为 {totalIfNum},按照IF-TYPE种类将其划分每个类别的IF判断数据如下：")
    title = ["  IF类别检查名称  | ", "检查出的个数 |", "类型占比"]
    print("{:<15} {:<10} {:<5}".format(*title))
    ifTypeLen = len(allFileIfTypeList)
    for k in range(ifTypeLen):
        ration = GetRatio(allFileIfTypeList[k], totalIfNum)
        ration = str(ration) + "%"
        print(f"++.\t{IfTypeInfo[k]}\t{allFileIfTypeList[k]}\t    {ration}")

    # 其他Trance的个数
    otherTranceNum = 0
    for otherItem in otherTraceList:
        otherTranceNum += otherItem[1]
    print(f'+-+-+跟踪开关的判断补充说明-+-当前判断宏为：TRACE_PROC 和 ORIGIN_TRACE_PROC+-+')
    print(f">> IF(Trance)的个数 {allFileIfTypeList[2]}, 宏(Trance)的个数 {otherTranceNum}")
    trcNumRation = round((allFileIfTypeList[2] + otherTranceNum) / (otherTranceNum + totalIfNum), 3)
    print(f"++. IF(Trance) + 宏(Trance) / 宏(Trance) + TOTAL-IF = {trcNumRation:.3%}")
    print(f'+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-++-+-+-+-+-+')


# 统计FILE-TYPE的详情
def StatisticsFileList(fileList):
    # 对获取的文件列表信息做统计
    fileTypeList = [0] * 5  # 0:.c文件; 1:.h文件; 2:TXT文件; 3:OTHER; 4:TotalFileNum
    for item in fileList:
        if item.endswith(".c"):
            fileTypeList[0] += 1
        elif item.endswith(".h"):
            fileTypeList[1] += 1
        elif item.endswith(".txt"):
            fileTypeList[2] += 1
        fileTypeList[4] += 1
    fileTypeList[3] = fileTypeList[4] - fileTypeList[0] - fileTypeList[1] - fileTypeList[2]
    return fileTypeList


#  文件目录处理类
class FileDirectoryProcessing:
    def __init__(self, filePath):
        self.filePath = filePath

    def ProcessFileDir(self, Path, fileList):
        fileDir = os.listdir(Path)
        for fileName in fileDir:
            filePath = os.path.join(Path, fileName)
            # 判断是否为文件
            if os.path.isfile(filePath):
                fileList.append(filePath)
            # 判断是否为目录
            elif os.path.isdir(filePath):
                self.ProcessFileDir(filePath, fileList)

    def GetFileRootPathList(self):
        fileList = []
        if os.path.isfile(self.filePath):
            fileList.append(self.filePath)
        elif os.path.isdir(self.filePath):
            self.ProcessFileDir(self.filePath, fileList)
        else:
            print('Error: Invalid Path!')
        return fileList


# 按照FILE类型累积IF-TYPE的数量
def UpdateIfNumByFileType(subFile, matchesLen, fileTypeOnIfNumList):
    if subFile.endswith(".c"):
        fileTypeOnIfNumList[0] += matchesLen
    if subFile.endswith(".h"):
        fileTypeOnIfNumList[1] += matchesLen
    fileTypeOnIfNumList[3] += matchesLen
    fileTypeOnIfNumList[2] = fileTypeOnIfNumList[3] - fileTypeOnIfNumList[1] - fileTypeOnIfNumList[0]


# 在strList中查找ifStrText是否存在
def searchSubStrExist(ifStrText, strList):
    strLen = len(strList)
    if strLen == 0:
        return 0
    for i in range(strLen):
        if ifStrText.find(strList[i]) != -1:
            return 1
    return 0


# 返回值检查
def CheckReturnValue(ifStrText):
    pattern = r"[\w]\s*!=\s*L2_OK"
    match = re.search(pattern, ifStrText, re.IGNORECASE)
    if match:
        # print("Check-Ret  " + ifStrText)
        return 1
    return 0


# 指针判空
def CheckPointerNull(ifStrText):
    # 指针判空检查宏
    pnj = ["IS_PTR_INVALID"]
    ret = searchSubStrExist(ifStrText, pnj)
    # if ret == 1:
    #     print("Check-Pointer  " + ifStrText)

    return ret


# 检查跟踪开关
def CheckTracingSwitch(ifStrText):
    # 跟踪项检查宏
    taj = ["IS_TRACE_ON", "TRACE_PROC", "IS_TRACE_ON", "IS_TRACE_OFF", "ORIGIN_IS_TRACE_ON", "ORIGIN_IS_TRACE_OFF",
           "ORIGIN_TRACE_PROC"]
    ret = searchSubStrExist(ifStrText, taj)
    if ret == 1:
        # print("Check-Trace  " + ifStrText)
        return ret
    # 处理将a=IS_TRACE_ON(),IF(a)的情况
    pattern = r"[\w]\(.*Tr.*On\)"
    match = re.search(pattern, ifStrText, re.IGNORECASE)
    if match:
        # print("Check-Trace  " + ifStrText)
        return 1

    pattern = r"[\w]\(.*trcSw\)"
    match = re.search(pattern, ifStrText, re.IGNORECASE)
    if match:
        # print("Check-Trace  " + ifStrText)
        return 1
    return 0


# 强值类型检查
def CheckStronglyType(ifStrText):
    # 强值类型检查宏
    stc = ["IS_USRID_INVALID", "IS_CELLINDEX_INVALID", "IS_TRPINDEX_INVALID", "IS_REAL_TRPINDEX", "IS_TRPPOS_INVALID",
           "IS_NRDUCELLID_INVALID", "IS_NRTRPID_INVALID", "IS_L1TRPID_INVALID", "IS_CD_SSBFREQINDEX_INVALID",
           "IS_MAX_SSBFREQINDEX_INVALID", "L2INF_SPECGETUSERLIMIT"]

    ret = searchSubStrExist(ifStrText, stc)
    # if ret == 1:
    #     print("StronglyValue  " + ifStrText)
    return ret


# 大颗粒特性检查
def CheckLargeFeature(ifStrText):
    pattern = r'\bF_DELETE\((.*?[\s\r\n]*)\)'
    if re.search(pattern, ifStrText, re.DOTALL):
        # print("Check-LargeFeather  " + ifStrText)
        return 1

    pattern = r'\bF_NON_DELETE\((.*?[\s\r\n]*)\)'
    if re.search(pattern, ifStrText, re.DOTALL):
        # print("Check-LargeFeather  " + ifStrText)
        return 1
    return 0


# 关系运算
def CheckNumLogic(ifStrText):
    regList = [r'\b^[\w]+(?:->[\w]+)*\s*(?:>|>=|<|<=|==|!=)\s*[\w]+$',  # 对象值->属性 >|>=|<|<=|==|!= 值 的结构
               r'\b^[\w]+\s+(?:>|>=|<|<=|==|!=)\s+(\b.*[\w](?:->[\w])?\b)$',  # 值 >|>=|<|<=|==|!= 对象值->属性 的结构
               r'\b.*?([\w]->[\w]*)\s+(?:>|>=|<|<=|==|!=)\s+.*?([\w]->[\w]*)',  # 对象->属性 >|>=|<|<=|==|!= 对象->属性 的结构
               r'\b\w+\s+(?:>|>=|<|<=|==|!=)\s+\w+\b'  # 值 >|>=|<|<=|==|!= 值 的结构
               ]

    regListLen = len(regList)
    for i in range(regListLen):
        if re.search(regList[i], ifStrText, re.S):
            # print("Check-Logic  " + ifStrText)
            return 1
    return 0


# if(func)函数式判断语句
def CheckFuncJudgment(ifStrText):
    pattern = r'\b[\s]*[\w]+\((.*?[\s\r\n]*)\)[\s]*'
    if re.search(pattern, ifStrText, re.DOTALL):
        # print("Check-Func  " + ifStrText)
        return 1
    return 0


# 文件中其他Trace标记统计
def OtherTranceFlag(fileText):
    # 其他跟踪项的宏
    fieldRegexList = [r'\bTRACE_PROC[\s]*\((.*?[\s\r\n]*)\)[\s]*',
                      r'\bORIGIN_TRACE_PROC[\s]*\((.*?[\s\r\n]*)\)[\s]*']
    otherTranceNum = 0
    for item in fieldRegexList:
        regex = re.compile(item, re.DOTALL)
        matches = regex.findall(fileText)
        if not matches or len(matches) == 0:
            continue
        otherTranceNum += len(matches)
        matches.clear()
    return otherTranceNum


# other-if判断
def OtherType(subIfStr):
    return 1


# 利用正则表达式统计FileList中IF相关信息
def CheckAndFillIfTypeNum(ifStrText, subFileIfTypeList):
    checkIfTypeFun = [CheckReturnValue,  # if(ret != L2_OK)
                      CheckPointerNull,  # if(指针判空)
                      CheckTracingSwitch,  # if(Trance)跟踪判断
                      CheckStronglyType,  # if(强值类型判断)比如用户ID，小区ID的上限判断
                      CheckLargeFeature,  # if(大颗粒特性判断)
                      CheckNumLogic,  # if(a>b)样式 值 > 值 结构
                      CheckFuncJudgment,  # if(func())函数式判断语句
                      OtherType]  # otherType:形如if(特性开关）
    funcLen = len(checkIfTypeFun)
    for i in range(funcLen):
        if checkIfTypeFun[i](ifStrText) == 1:
            subFileIfTypeList[i] = subFileIfTypeList[i] + 1
            break


# 利用正则表达式统计FileList中IF相关信息
def CheckAndFillIfTypeNumAndExcTime(ifStrText, subFileIfTypeList, subFileExcTime, excTime):
    checkIfTypeFun = [CheckReturnValue,  # if(ret != L2_OK)
                      CheckPointerNull,  # if(指针判空)
                      CheckTracingSwitch,  # if(Trance)跟踪判断
                      CheckStronglyType,  # if(强值类型判断)比如用户ID，小区ID的上限判断
                      CheckLargeFeature,  # if(大颗粒特性判断)
                      CheckNumLogic,  # if(a>b)样式 值 > 值 结构
                      CheckFuncJudgment,  # if(func())函数式判断语句
                      OtherType]  # otherType:形如if(特性开关）
    funcLen = len(checkIfTypeFun)
    for i in range(funcLen):
        if checkIfTypeFun[i](ifStrText) == 1:
            subFileIfTypeList[i] += 1
            subFileExcTime[i] += int(excTime)
            break


#  文件处理IF类
def AccumulateFileIfType(subFileIfTypeList, allFileIfTypeList):
    allIfTypeNumListLen = len(allFileIfTypeList)
    if allIfTypeNumListLen == 0:
        return
    for i in range(allIfTypeNumListLen):
        allFileIfTypeList[i] += subFileIfTypeList[i]


class StaticAnalysisIf:
    def __init__(self, fileList):
        self.fileList = fileList

    # 统计文件中其他的跟踪标记
    # 参数1：不同文件类型中的IF数目、参数2：按IF-TYPE划分的IF数目、参数3：每个文件中的IF数目、4、额外的Trace标记计数
    def GetIfInfoByRegularity(self, fileTypeOnIfNumList, allFileIfTypeList, eachFileIfNum, otherTraceList):
        eachFileIfType = []
        for subFile in self.fileList:
            subFileIfTypeList = [0] * IF_TYPE_NUM
            with open(subFile, 'r', encoding='utf-8') as file:
                try:
                    fileText = file.read()
                    # field_regex = r'\bif[\s]*\((.*?[\s\r\n]*)\)[\s]*\{'  # 匹配源文件中全部IF关键字
                    field_regex = r'\bif\s*\(([^{}]+)\)\s*\{'
                    regex = re.compile(field_regex, re.DOTALL)
                    matches = regex.findall(fileText)
                    if not matches:
                        continue
                    matchesLen = len(matches)  # 当前文件的IF判断条数
                    eachInfo = [[subFile, str(matchesLen)]]
                    eachFileIfNum.append(eachInfo)
                    UpdateIfNumByFileType(subFile, matchesLen, fileTypeOnIfNumList)
                    # print(f"文件 {subFile} 中共匹配到 {matchesLen} 个 IF 条件语句：")
                    for idx in range(matchesLen):  # 逐一处理筛选出来的IF判断语句体
                        # print(f"{idx + 1}: {matches[idx]}")
                        CheckAndFillIfTypeNum(matches[idx], subFileIfTypeList)

                    # 保存当前FILE不同IF-TYPE的信息
                    eachFileIfType.append([subFile, matchesLen, subFileIfTypeList])
                    matches.clear()  # 清空筛选列表
                    otherTranceNum = OtherTranceFlag(fileText)  # 统计其他的Trance标记
                    if otherTranceNum != 0:
                        otherTraceList.append([subFile, otherTranceNum])
                finally:
                    file.close()
            # 将单个FILE的IF-TYPE累加到总list中
            AccumulateFileIfType(subFileIfTypeList, allFileIfTypeList)

        # 将所有FILE的IF-TYPE信息保存到Excel中
        WriteAllFileIfTypeToExcel(eachFileIfType)


#  获得待码覆盖率网页中的行号
def GetLineNum(lineStr):
    if lineStr.isspace() or len(lineStr) == 0:
        return 0
    lineNum = re.findall(r'\d+', lineStr)
    return lineNum[0]


# 输入列表返回if(){}中的 { 的所在位置下标
def GetIfLeftBraceIndex(if_item):
    for index, item in enumerate(if_item):
        if "{" in item:
            return index
    return -1


#  获得待码覆盖率网页中的执行次数
def GetExcTime(timeStr):
    if timeStr.isspace() or len(timeStr) == 0:
        return 0
    excTime = re.sub(r'\s+', '', timeStr)  # 将多个连续的空格替换为一个空格
    excTime = re.findall(r'\d+', excTime)  # 提取出数字
    return excTime[0]


#  获得待码覆盖率网页中的执行次数
def GetIfJudgeItemExcTime(if_Txt):
    ifList = if_Txt.split('\n')
    ifListLen = len(ifList)
    rowIndex = 0
    while rowIndex < ifListLen:
        item = ifList[rowIndex]
        itemList = item.split(':')
        if GetIfLeftBraceIndex(itemList) != -1:
            # print(itemList)
            # 单独处理IF语句体的行号
            rowIndex += 1
            while rowIndex < ifListLen:
                subItemList = ifList[rowIndex].split(":")
                subItemListLen = len(subItemList)
                if subItemListLen < 2:
                    rowIndex += 1
                    continue
                excTime = subItemList[1]
                excTime = re.sub(r'\s+', '', excTime)
                excTime = re.findall(r'\d+', excTime)
                # print(f"excTime = {excTime}")
                if len(excTime) == 0:
                    rowIndex += 1
                else:
                    return excTime[0]
        rowIndex += 1
    return -1


#  获得IF()内容
def GetIfJudgeContent(i, if_item2len, if_item2):
    ifTxtContent = ''
    while i < if_item2len:
        strTxt = if_item2[i]
        # 寻找换行符 \n 如存在则保留\n之前的内容
        n_index = strTxt.find('\n')  # 找到\n的位置
        if n_index != -1:
            strTxt = strTxt[:n_index] + ' '
        # 将多个连续的空格替换为空字符
        strTxt = re.sub(r'\s+', ' ', strTxt)
        # 如果strTxt是纯数字则替换为空字符
        if re.fullmatch(r'\d+', strTxt):
            strTxt = re.sub(r"\b\d+(?!\))\b", ' ', strTxt)  # 如果是xx)这样的则不替换，含义是判断语句中有数字
        # 将大括号{后所有的字符替换为空字符
        strTxt = re.sub(r'{\s*.*', '', strTxt)
        if len(strTxt) == 0 or strTxt.isspace():
            i += 1
            continue
        ifTxtContent += strTxt
        i += 1

    ifTxtContent = re.sub(r'\s+', ' ', ifTxtContent)  # 将多个连续的空格替换为空字符
    return ifTxtContent


#  每个文件的IF种类和执行次数累加
def UpdateIfTypeCountAndExcTime(subFileIfTypeList, EachFileIfTypeList, SubIfExcTime, EachFileIfTypeExcTime):
    sumIfCount = 0
    sumIfExc = 0
    for i in range(IF_TYPE_NUM):
        EachFileIfTypeList[i] += subFileIfTypeList[i]
        sumIfCount += subFileIfTypeList[i]
        EachFileIfTypeExcTime[i] += SubIfExcTime[i]
        sumIfExc += SubIfExcTime[i]
    EachFileIfTypeList[IF_TYPE_NUM] += sumIfCount
    EachFileIfTypeExcTime[IF_TYPE_NUM] += sumIfExc


#  正则表达是直接匹配IF判断语句，不关心IF判断语句的内容，直接获得IF的行数和执行次数
#  参数说明：1、循环语句体用来判断该语句体内是否有IF判断语句
#          2、循环语句体的信息loopInfoList 中每个条目：[循环的行数、循环执行的次数、循环内IF的个数 N，[IF的行数，IF的执行次数]*N]
#          3、该FILE内的所有循环信息：TotalLoop中每个条目：loopInfoList
def GetIfLineAndExcTimeWithoutIfContent(loop_txt, loopInfoList, TotalLoop):
    ifReg = r'\d+([\s]*.*?)\bif\b'
    if_matches = re.finditer(ifReg, loop_txt)
    if not if_matches:
        return
    ifCount = 0
    ifInfoList = []  # 该循环语句体内的IF判断信息，假设有N个，(IF的行数，IF执行次数) * N
    for im in if_matches:
        if_Txt = im.group(0)
        if_item = if_Txt.split(':')
        # print(f"f++ {if_item}")
        if_LineNum = GetLineNum(if_item[0])
        if_ExcTime = GetExcTime(if_item[1])
        ifCount += 1
        # print(f" FIRST  if line ={if_LineNum}, time = {if_ExcTime}")
        ifInfoList.append([if_LineNum, if_ExcTime])

    loopInfoList.append(ifCount)
    loopInfoList.append(ifInfoList)
    TotalLoop.append(loopInfoList)


#  正则表达是直接匹配IF判断语句，获得IF判断语句的内容，获得IF的行数和执行次数以及IF判断语句体的类型
#  参数说明：1、循环语句体用来判断该语句体内是否有IF判断语句
#          2、循环语句体的信息loopInfoList 中每个条目：[循环的行数、循环执行的次数、循环内IF的个数 N，[IF的行数，IF的执行次数]*N]
#          3、该FILE内的所有循环信息：TotalLoop中每个条目：loopInfoList
def GetIfLineAndExcTimeWithIfContent(loop_txt, loopInfoList, TotalLoop, EachFileIfTypeList, EachFileIfTypeExcTime):
    ifReg = r'\d+([\s]*.*?)\bif\s*\(([^{}]+)\)\s*\{'
    if_matches = re.finditer(ifReg, loop_txt)
    if not if_matches:
        return
    ifCount = 0
    ifInfoList = []  # 该循环语句体内的IF判断信息，假设有N个，(IF的行数，IF执行次数) * N
    subFileIfTypeList = [0] * IF_TYPE_NUM  # IF-TYPE种类的集合
    SubIfExcTime = [0] * IF_TYPE_NUM  # IF-TYPE的执行数的集合
    for im in if_matches:
        if_Txt = im.group()
        if_item = if_Txt.split(':')
        # print(f"s++ {if_item}")
        if_itemLen = len(if_item)
        if_LineNum = GetLineNum(if_item[0])
        if_ExcTime = GetExcTime(if_item[1])
        startIndex = 2
        ifTxtContent = GetIfJudgeContent(startIndex, if_itemLen, if_item)
        # print(ifTxtContent)
        # 需要判断IF的判断种类
        CheckAndFillIfTypeNumAndExcTime(ifTxtContent, subFileIfTypeList, SubIfExcTime, if_ExcTime)
        ifCount += 1
        # print(f" SECOND  if line ={if_LineNum}, time = {if_ExcTime}")
        ifInfoList.append([if_LineNum, if_ExcTime])
    UpdateIfTypeCountAndExcTime(subFileIfTypeList, EachFileIfTypeList, SubIfExcTime, EachFileIfTypeExcTime)
    # print(subFileIfTypeList, SubIfExcTime)
    loopInfoList.append(ifCount)
    loopInfoList.append(ifInfoList)
    TotalLoop.append(loopInfoList)


def ProcessRegularExpr(loopReg, txtContent, EachFileIfTypeList, EachFileIfTypeExcTime):
    loop_matches = re.finditer(loopReg, txtContent)
    if not loop_matches:
        return None
    TotalLoop = []
    for match in loop_matches:
        loopInfoList = []
        loop_txt = match.group()
        loop_item = loop_txt.split(':')
        loop_LineNum = GetLineNum(loop_item[0])
        loop_ExcTime = GetExcTime(loop_item[1])
        # print(f"loop line = {loop_LineNum} time = {loop_ExcTime}")
        loopInfoList.append(loop_LineNum)
        loopInfoList.append(int(loop_ExcTime))
        # 这个两个函数使用的IF正则匹配有差异一个不带IF判断语句体，一个待IF判断语句体，目前需要待IF判断语句体的
        # GetIfLineAndExcTimeWithoutIfContent(loop_txt, loopInfoList, TotalLoop)
        GetIfLineAndExcTimeWithIfContent(loop_txt, loopInfoList, TotalLoop, EachFileIfTypeList, EachFileIfTypeExcTime)
    return TotalLoop


# 将每个FILE中LOOP内IF的行号+执行次数+IF个数信息写入TXT
def WriteLoopInfoToTxt(filepath, resultList):
    fileName = r'D:/IfInLoopType_EachFile_CoverageAnalysis.txt'
    RemoveFileByName(fileName)
    with open(fileName, 'a') as file:
        try:
            file.writelines(f'+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-++-+-+-+-+-+\n')
            file.writelines(f'>> 在当前文件目录下 {filepath} 中各个文件的代码覆盖率详情如下\n')
            for sabres in resultList:
                subResLen = len(sabres)
                if subResLen <= 1:
                    continue
                for i in range(subResLen):
                    if i == 0:
                        loop = i + 1
                        loopCount = 0
                        while loop < subResLen:
                            loopCount += len(sabres[loop])
                            loop += 1
                        file.writelines(f'  文件名称 {sabres[0]} 中循环个数 {loopCount} \n')
                    else:
                        loopInfo = sabres[i]
                        loopInfoLen = len(loopInfo)
                        for k in range(loopInfoLen):
                            item = loopInfo[k]
                            file.writelines(f'    {k}. 在 {item[0]} 行存在循环，该循环执行次数 {item[1]}，循环中IF个数{item[2]} \n')
                            ifCount = item[2]
                            if ifCount == 0:
                                continue
                            for j in range(ifCount):
                                file.writelines(f'        在 {item[3][j][0]} 行存在 IF 判断，该循环执行次数{item[3][j][1]}\n')

        finally:
            file.close()


# 将每个FILE中的IF-TYPE的[个数、个数占比，执行次数，次数占比]信息写入到Excel中
# 同时记录获取代码覆盖率失败和成功的FILE
def WriteLoopIfTypeToExcel(TotalFileIfTypeListAndExcTimes, failedList, successList):
    fileName = r'D:/IfInLoopType_EachFile_Analysis.xlsx'
    RemoveFileByName(fileName)
    IfTypeInfo = ["返回值检查", "指针判空检查", "跟踪开关检查", "强值类型判断", "大颗粒特性判断", "关系运算判断", "函数式判断语句",
                  "OTHER类型"]
    writer = pd.ExcelWriter(fileName, engine='xlsxwriter')
    # 输出FILE-TYPE信息
    IfTypeRows = []
    Title = ['文件名称', 'IF类别检查名称', 'IF检查出的个数', 'IF检查个数占比', 'IF执行次数', 'IF执行次数占比']
    for item in TotalFileIfTypeListAndExcTimes:
        last_name = item[0].split("\\")[-1]
        ifTypeList = item[1]
        ifExcList = item[2]
        for i in range(IF_TYPE_NUM):
            ration1 = GetRatio(ifTypeList[i], ifTypeList[IF_TYPE_NUM])
            ration2 = GetRatio(ifExcList[i], ifExcList[IF_TYPE_NUM])
            IfTypeRows.append([last_name, IfTypeInfo[i], ifTypeList[i], ration1, ifExcList[i], ration2])
        IfTypeRows.append(['小计', 'IF-TYPE', ifTypeList[IF_TYPE_NUM], 1, ifExcList[IF_TYPE_NUM], 1])

    df = pd.DataFrame(IfTypeRows, columns=Title)
    df.to_excel(writer, sheet_name="每个文件内循环中IF判断种类和执行次数汇总分析", index=False)

    Title1 = ['该FILE无覆盖网页在[5g.rnd.huawei.com]中']
    df1 = pd.DataFrame(failedList, columns=Title1)
    df1.to_excel(writer, sheet_name="文件无对应的代码覆盖率-汇总分析", index=False)

    Title2 = ['该FILE有覆盖网页在[5g.rnd.huawei.com]中']
    df2 = pd.DataFrame(successList, columns=Title2)
    df2.to_excel(writer, sheet_name="文件对应的代码覆盖率-汇总分析", index=False)

    writer.save()


# 将代码覆盖率网页中的IF行数、执行次数、IF内容等信息写入到Excel中
def WriteAllCodeIfInfoToExcel(resultList, failedList):
    fileName = r'D:/IF_Information_Details.xlsx'
    RemoveFileByName(fileName)
    writer = pd.ExcelWriter(fileName, engine='xlsxwriter')

    IfTypeRows = []
    Title = ['文件名称', 'IF所在行号', 'IF执行次数', 'IF语句体执行次数', 'IF判断所属类型', 'IF判断所属子类型', 'IF判断表达式',
             '解决方案', '覆盖率链接']
    for item in resultList:
        last_name = item[0].split("\\")[-1]
        ifInfo = item[1]
        url = item[2]
        ifInfoLen = len(ifInfo)
        for i in range(ifInfoLen):
            IfTypeRows.append([last_name, ifInfo[i][0], ifInfo[i][1], ifInfo[i][2], '', '', ifInfo[i][3], '', url])
    df = pd.DataFrame(IfTypeRows, columns=Title)
    df.to_excel(writer, sheet_name="每个FILE中IF关键字的总明细", index=False)

    Title1 = ['该FILE无覆盖网页在[5g.rnd.huawei.com]中']
    df1 = pd.DataFrame(failedList, columns=Title1)
    df1.to_excel(writer, sheet_name="文件无对应的代码覆盖率-汇总分析", index=False)

    writer.save()


# 根据URL获得该HTML中的内容
def getHtmlTxtByUrl(url, header):
    txtContent = ""
    res = requests.get(url=url, headers=header, verify=False)
    if res.status_code != 200:
        print(f"url22 = {url}")
        return txtContent
    # 按照标签提取内容
    soup = BeautifulSoup(res.content, 'html.parser')
    span_tags = soup.find_all('pre', class_=['source'])
    for item in span_tags:
        txtContent += item.text
    return txtContent


#  通过文件名称获得->该FileName的代码覆盖率URL->访问URL获得该HTMl的内容
def GetHtmlContentByFilePath(filePath, failedList, successList):
    params = {
        "repo_path": REPO_PATH,
        "src_path": SRC_PART1 + filePath,
    }
    response = requests.get(url=URL, params=params, headers=headers, verify=False)
    if response.status_code != 200:
        failedList.append(f"该文件获取代码覆盖率链接失败!! {filePath} \n")
        return None
    url = response.json()['itst_url']

    txtContent = getHtmlTxtByUrl(url, headers)
    if len(txtContent) == 0:
        failedList.append(f"无法获取该HTMl的内容 {url} \n")
        return None
    # print(url)
    successList.append(f"{url}\n")
    return txtContent


#  获得每个FILE中循环内IF判断的信息
def GetLoopInnerIfInfo(subFile, txtContent, TotalFileIfTypeListAndExcTimes, resultList):
    # 正则表达式来匹配C语言中的循环[for\while\do-while]
    regList = [
        r'\d+([\s]*.*?)\bfor\b[\s]*\((.*?[\s\r\n]*)\)[\s]*\{(([\s\S]*?)(?:[^{}]*(?:\{([\s\S]*?)[^{}]*\}))*[^{}]*?)\}',
        r'\d+([\s]*.*?)\bwhile\b[\s]*\((.*?[\s\r\n]*)\)[\s]*\{(([\s\S]*?)(?:[^{}]*(?:\{([\s\S]*?)[^{}]*\}))*[^{}]*?)\}',
        r'\d+([\s]*.*?)\bdo\b[\s]*\{(([\s\S]*?)(?:[^{}]*(?:\{([\s\S]*?)[^{}]*\}))*[^{}]*?)\}[\s]*while[\s]*([\s\S]*?)'
    ]
    retList = [subFile]
    # IF的判断种类和IF的执行次数lIST
    EachFileIfTypeList = [0] * (IF_TYPE_NUM + 1)  # [IF_TYPE*(8 + 1), SUM(IF_TYPE*(8 + 1)))]
    EachFileIfTypeExcTime = [0] * (IF_TYPE_NUM + 1)  # [IF_EXC-TIME*(8 + 1), SUM(IF_EXC-TIME*(8 + 1))]
    for loopReg in regList:
        ret = ProcessRegularExpr(loopReg, txtContent, EachFileIfTypeList, EachFileIfTypeExcTime)
        if len(ret) != 0:
            retList.append(ret)
    # print(f"++= {EachFileIfTypeList}{EachFileIfTypeExcTime}")
    # 将每个文件的LOOP-IF种类划分信息汇聚保存
    if EachFileIfTypeList[IF_TYPE_NUM] != 0:
        TotalFileIfTypeListAndExcTimes.append([subFile, EachFileIfTypeList, EachFileIfTypeExcTime])

    resultList.append(retList)


# 获取代码覆盖率网页上的IF的行号：执行次数：IF判断语句
def GetFileALLIfInfoWithUrl(txtContent):
    ifInfoList = []  # [IF行号，IF执行次数，IF语句体执行次数，IF语句判断内容]
    # ifReg = r'\d+([\s]*.*?)\bif\s*\(([^{}]+)\)\s*\{'
    ifReg = r'\d+([\s]*.*?)\bif\s*\(([^{}]+)\)\s*\{(([\s\S]*?)(?:[^{}]*(?:\{([\s\S]*?)[^{}]*\}))*[^{}]*?)\}'
    if_matches = re.finditer(ifReg, txtContent)
    if not if_matches:
        return ifInfoList

    for im in if_matches:
        if_Txt = im.group()
        # print(f"if_TXT\n {if_Txt}")
        if_item = if_Txt.split(':')
        # print(f"if_item\n {if_item}")
        if_itemLen = len(if_item)
        if_LineNum = GetLineNum(if_item[0])
        if_ExcTime = GetExcTime(if_item[1])
        # if () { }中的 { 的下标所在的位置
        ifLeftBrace = GetIfLeftBraceIndex(if_item)
        startIndex = 2  # 列表开始处理的位置
        endIndex = min(ifLeftBrace, if_itemLen) + 1  # 列表结束处理的位置
        # 获得IF判断体的内容
        ifTxtContent = GetIfJudgeContent(startIndex, endIndex, if_item)
        # 获得IF语句体的执行次数
        ifJudgeItemExcTime = GetIfJudgeItemExcTime(if_Txt)
        # print(f"ifLeftBrace= {ifLeftBrace}, startIndex = {startIndex}, endIndex = {endIndex}")
        # print(f"IF Line = {if_LineNum}, Time = {if_ExcTime}, IfExpression={ifTxtContent},"
        #       f" ifJudgeItemExcTime = {ifJudgeItemExcTime}")
        ifInfoList.append([if_LineNum, int(if_ExcTime), int(ifJudgeItemExcTime), ifTxtContent])

    # print(ifInfoList)
    return ifInfoList


#  直接通过爬虫获取代码覆盖率的原始HTML信息，处理里面的数据
def ProcessCodeCoverage(filepath, filePathList, TotalFileIfTypeListAndExcTimes):
    # 对resultList的解析:  filePathList中共有文件K个，resultList =[item*K],其中每个item组成如下
    # item：M个循环以及每个循环下的N的IF情况，[文件名称，[[循环的行数、循环执行的次数、循环内IF的个数 N，[IF的行数，IF的执行次数]*N]*M]]
    resultList = []
    failedList, successList = [], []  # 获取覆盖率失败的文件记录 / 获取覆盖率成功的文件记录
    for subFile in filePathList:
        filePath = filePathTranslation(subFile)
        txtContent = GetHtmlContentByFilePath(filePath, failedList, successList)
        if txtContent is None:
            continue
        GetLoopInnerIfInfo(subFile, txtContent, TotalFileIfTypeListAndExcTimes, resultList)

    # 将每个文件的M*N的情况记录到TXT中
    WriteLoopInfoToTxt(filepath, resultList)
    # 将每个FILE中的IF-TYPE个数以及占比和IF-TYPE执行次数以及占比记录到Excel中
    WriteLoopIfTypeToExcel(TotalFileIfTypeListAndExcTimes, failedList, successList)


def GetUrlByFilePath(filePath, failedList):
    params = {
        "repo_path": REPO_PATH,
        "src_path": SRC_PART1 + filePath,
    }
    response = requests.get(url=URL, params=params, headers=headers, verify=False)
    if response.status_code != 200:
        failedList.append(f"该文件获取代码覆盖率链接失败!! {filePath} \n")
        return None
    url = response.json()['itst_url']
    print(url)
    return url


def GetHtmlTxtByUrl(url, failedList):
    txtContent = getHtmlTxtByUrl(url, headers)
    if len(txtContent) == 0:
        failedList.append(f"无法获取该HTMl的内容 {url} \n")
        return None
    return txtContent


#  直接通过爬虫获取代码覆盖率的原始HTML信息，处理里面的数据
def GetAllIfInfoWithCodeCoverage(filePathList):
    # 对resultList的解析:  filePathList中共有文件K个，resultList =[item*K],其中每个item组成如下
    # item：[文件名称，[IF的行数，IF的执行次数，IF判断语句内容]*N]，URL链接]
    resultList = []
    failedList = []  # 获取覆盖率失败的文件记录
    for subFile in filePathList:
        filePath = filePathTranslation(subFile)

        url = GetUrlByFilePath(filePath, failedList)
        if url is None:
            continue
        txtContent = GetHtmlTxtByUrl(url, failedList)
        if txtContent is None:
            continue

        item = GetFileALLIfInfoWithUrl(txtContent)
        if len(item) != 0:
            resultList.append([subFile, item, url])

    # 将FILE的IF信息保存到Excel中
    WriteAllCodeIfInfoToExcel(resultList, failedList)


# 通过文件名称->URL->HTML->if判断信息
def GetIfCodeCoverageDetailFormOnLine():
    print(f">> 请输入要分析的文件路径[单个文件或文件目录均可]")
    file_path = str(input())
    print(f">> 你输入的内容是：" + file_path + " 对输入的内容针对 [IF关键字] 的分析详情如下:")
    #  文件目录处理：获得FILE路径列表和FILE种类列表
    fileDirProcess = FileDirectoryProcessing(file_path)
    filePathLists = fileDirProcess.GetFileRootPathList()
    fileTypeLists = StatisticsFileList(filePathLists)  # FILE-TYPE作为统计输出使用
    #  逐一处理FILE，按照既定的IF种类划分分类
    if len(filePathLists) == 0:
        print(f"你输入的{file_path}下没有合法的文件路径，请检查！！！ ")
    # 对文件列表中的文件逐一进行IF信息处理
    # staticAnalysisIf = StaticAnalysisIf(filePathLists)
    # fileTypeOnIfNumLists = [0] * 4  # 不同文件类型下IF判断的数目[.c,.h,Other,Total]
    # allFileIfTypeLists = [0] * IF_TYPE_NUM  # 按照划分的IF-TYPE统计数目
    # eachFileIfNums = []  # 每个文件的IF不同数目 [文件名称, IF-NUM]
    # otherTraceLists = []  # [文件名称, OtherTranceNum]
    # staticAnalysisIf.GetIfInfoByRegularity(fileTypeOnIfNumLists, allFileIfTypeLists, eachFileIfNums, otherTraceLists)
    # PrintIfStatistInfo(fileTypeLists, fileTypeOnIfNumLists, allFileIfTypeLists, otherTraceLists, file_path)

    # 直接通过FILE-NAME获取该FILE的覆盖率网页直接处理
    # TotalFileIfTypeListAndExcTime = []
    # ProcessCodeCoverage(file_path, filePathLists, TotalFileIfTypeListAndExcTime)
    GetAllIfInfoWithCodeCoverage(filePathLists)
    # 将IF统计信息输出为Excel
    # WriteStaticIfInfoToExcel(TotalFileIfTypeListAndExcTime, fileTypeLists, fileTypeOnIfNumLists, allFileIfTypeLists,
    #                          otherTraceLists, file_path)


def GetLocalFile():
    print(f">> 请输入要分析的文件名称")
    filePath = str(input())
    print(f">> 你输入的内容是：" + filePath + " 对输入的内容针对 [IF关键字] 的分析详情如下:")
    listUrl = []
    with open(filePath, "r", encoding='utf-8-sig') as f:
        try:
            listUrl = f.read().splitlines()
        finally:
            f.close()
    return listUrl


def GetUrlFormLocalFile():
    listUrl = GetLocalFile()
    if len(listUrl) == 0:
        print(f"输入的文件中无内容，请核对和在输入！！")
        return
    print(f"待处理的文件总数: {len(listUrl)}")

    # 对resultList的解析:  filePathList中共有文件K个，resultList =[item*K],其中每个item组成如下
    # item：[文件名称，[IF的行数，IF的执行次数，IF判断语句内容]*N]，URL链接]
    resultList = []
    failedList = []  # 获取覆盖率失败的文件记录
    # listUrl = [
    #     'https://webide-x86-sz17.starling.huawei.com:31731/ulsch/framework/usrmng/usrlistagg/src/ulsch_usrmng_usrgrp.c.gcov.html']
    for url in listUrl:
        txtContent = GetHtmlTxtByUrl(url, failedList)
        if txtContent is None:
            continue
        item = GetFileALLIfInfoWithUrl(txtContent)
        if len(item) != 0:
            last_name = url.split("/")[-1]
            last_name = last_name.rsplit('.', 2)[0]
            resultList.append([last_name, item, url])

    # 将FILE的IF信息保存到Excel中
    WriteAllCodeIfInfoToExcel(resultList, failedList)


# 静态代码分析-IF分析的主函数
if __name__ == '__main__':
    print('+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-++-+-+-+-+-+')
    print(f">> 请选择处理模式: 结果位置 D:IF_Information_Details.xlsx")
    print(f"    1: 输入文件名称或者目录读取 5g.rnd.huawei.com 中的代码覆盖率信息")
    print(f"    2: 输入自定义的TEST的代码覆盖率路径文件信息")
    print('+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-++-+-+-+-+-+')
    print(f">> 请输入选择： 输入 1 或 2 ")
    choosePattern = int(input())
    print('+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-++-+-+-+-+-+')
    if choosePattern == 1:
        GetIfCodeCoverageDetailFormOnLine()
    elif choosePattern == 2:
        GetUrlFormLocalFile()
    else:
        print(f"没有第三个选项 :)")
    input()
    print("\n >> 恭喜哦，程序已成功执行完毕 :) | Congratulations, the program is executed successfully :)")

```
