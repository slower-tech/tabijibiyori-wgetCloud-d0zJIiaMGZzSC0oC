
## **1\.****背景**


本qiang\~本周在处理手头项目工作的时候，遇到了一个问题，就是友方提供了一个公司名称列表(量不小\~，因此无法人工处理)，且该公司名称列表均为简称，需要与库中的全称做一个映射匹配。


看似简单的一个需求，但传统的技术手段貌似都无法派上用场，比如语义相似度，文本编辑距离等等。


因此本qiang花费了半天的时间思考并解决了该任务，遂将工作记录如下，且本着开放共享，将核心源码进行公开，欢迎讨论\~


## **2\.整体框架**


![](https://img2024.cnblogs.com/blog/602535/202411/602535-20241118180333359-1846367619.png)


 


 


其实，原理也非常简单，由于本地数据库缺乏公司的完整信息，但可以借助互联网资源来搜索公司的相关信息，比如官网介绍、天眼查等来源，然后将检索后的结果通过大模型自带的推理能力输出最终结果。


本文中使用的搜索引擎是duckduckgo\_search(需要kexue上网)，大模型调用使用的duckduckgo\_search内部集成的gpt\-4o\-mini（理论上只要能kexue上网，即可免费使用gpt\-4o\-mini）。


## 3\. **效果展示**




| AutoX | 深圳安途智行科技有限公司 |
| --- | --- |
| Cosmose | 翱觅苷（上海）信息科技有限公司 |
| Magic Data | 北京晴数智慧科技有限公司 |
| Minimax | 名之梦（上海）科技有限公司 |
| Momenta | 北京初速度科技有限公司 |
| Testin云测 | 北京云测信息技术有限公司 |
| 一流科技 | 一流科技有限公司 |
| 三六零 | 三六零安全科技股份有限公司 |
| 东杰智能 | 东杰智能科技集团股份有限公司 |
| 东软 | 东软集团股份有限公司 |
| 中心通讯 | 中兴通讯股份有限公司 |
| 中科创达 | 中科创达软件股份有限公司 |
| 中科曙光 | 曙光信息产业股份有限公司 |
| 中科视拓 | 中科视拓（北京）科技有限公司 |
| 中译语通 | 中译语通科技股份有限公司 |
| 九四智能 | 广州九四智能科技有限公司 |
| 九章云极 | 北京九章云极科技有限公司 |
| 云天励飞 | 深圳云天励飞技术股份有限公司 |
| 云徙科技 | 广州云徙科技有限公司 |
| 亚信科技 | 亚信科技控股有限公司 |


## 4\. **全部源码**


由于调用检索相对耗时，因此分为公司简称检索和公司全称提取两个模块


### **4\.1公司简称检索**


![](https://images.cnblogs.com/OutliningIndicators/ContractedBlock.gif)![](https://images.cnblogs.com/OutliningIndicators/ExpandedBlockStart.gif)


```
from duckduckgo_search import DDGS
import json
import time

def save_datas(file_path, datas, json_flag=True, all_flag=False, with_indent=False, mode='w'):
    """保存文本文件"""
    with open(file_path, mode, encoding='utf-8') as f:
        if all_flag:
            if json_flag:
                f.write(json.dumps(datas, ensure_ascii=False, indent= 4 if with_indent else None))
            else:
                f.write(''.join(datas))
        else:
            for data in datas:
                if json_flag:
                    f.write(json.dumps(data, ensure_ascii=False) + '\n') 
                else:
                    f.write(data + '\n')


def search_companies(companies):
    results = []
    for company in companies:
        if '公司' in company:
            results.append({
                'company': company,
                'search_results': 'company'
            })
            continue
        text = f'{company} 公司名全称'
        
        search_results = None
        while search_results is None:
            try:
                search_results = DDGS().text(text, max_results=10)
                if search_results: break
            except Exception as e:
                print('sleep 2s')
                time.sleep(2)
                continue
        results.append({
            'company': company,
            'search_results': search_results
        })
        time.sleep(2)
    save_datas('data/公司简称检索结果.json', results)


def get_datas(file_path, json_flag=True, all_flag=False, mode='r'):
    """读取文本文件"""
    results = []
    
    with open(file_path, mode, encoding='utf-8') as f:
        for line in f.readlines():
            if json_flag:
                results.append(json.loads(line))
            else:
                results.append(line.strip())
        if all_flag:
            if json_flag:
                return json.loads(''.join(results))
            else:
                return '\n'.join(results)
        return results
    

if __name__ == '__main__':
    search_companies(get_datas('data/公司简称列表.txt', json_flag=False))
```


View Code
 


**4\.2公司全名提取**


![](https://images.cnblogs.com/OutliningIndicators/ContractedBlock.gif)![](https://images.cnblogs.com/OutliningIndicators/ExpandedBlockStart.gif)


```
from duckduckgo_search import DDGS
import json
import time


PROMPT = """你是一个助手，你的任务是基于输入的公司名简称以及搜索结果，分析并提取出公司名简称对应的公司名全称。要求如下：
1. "简称"是公司名简称，"搜索结果"是基于互联网的搜索后的资源，需要根据"简称"和"搜索结果"进行分析，并输出公司全称，如果无法确认，请返回"无"；
2. 如果检索结果不包含公司全称，请基于你所学习的知识可以进一步判断; 
3. 输出结果只包含公司名的全称信息，且只能包含一个，不需要输出解释信息;
4. 输入的公司名简称均是科技领域的知名公司，这点请注意；

示例：
简称: 京东
搜索结果：
1. 京东（中国1998年创立的自营式电商企业）_百度百科\n京东（股票代码：jd），中国自营式电商企业，创始人刘强东初期担任京东集团董事局主席兼首席执行官，2021年9月，徐雷获任集团总裁。京东旗下设有京东商城、京东金融、拍拍网、京东智能、o2o及海外事业部等。1998年6月18日，刘强东在 中关村成立京东公司。
2. 京东集团 - 维基百科，自由的百科全书\n东集团. 京东集团 （NASDAQ： JD 、 港交所： 9618 、 港交所： 89618 （人民幣結算）），前稱 360buy 和 京東商城，由刘强东于1998年6月18日创立，是一家总部位于 北京 的 中国 电子商务公司，主要為 B2C 模式的購物網站 。. 2014年，京东集团在 美国 纳斯达克证券交易 ...
3. 京东集团股份有限公司 - 爱企查\n简介： 京东集团股份有限公司（JD.com, Inc.）于2006年11月6日在在英属维尔京群岛注册成立的公司，通过中国境内的子公司和VIE开展经营活动，公司总部位于北京。. 京东是专业的综合性网上购物商城，是中国B2C市场最大的3C网购专业平台，是中国电子商务领域最受 ...
输出: 京东集团股份有限公司

现在，请按照要求完成：
简称: {company_name}
搜索结果: {search_results}
输出: 
"""

def save_datas(file_path, datas, json_flag=True, all_flag=False, with_indent=False, mode='w'):
    """保存文本文件"""
    with open(file_path, mode, encoding='utf-8') as f:
        if all_flag:
            if json_flag:
                f.write(json.dumps(datas, ensure_ascii=False, indent= 4 if with_indent else None))
            else:
                f.write(''.join(datas))
        else:
            for data in datas:
                if json_flag:
                    f.write(json.dumps(data, ensure_ascii=False) + '\n') 
                else:
                    f.write(data + '\n')

def get_datas(file_path, json_flag=True, all_flag=False, mode='r'):
    """读取文本文件"""
    results = []
    
    with open(file_path, mode, encoding='utf-8') as f:
        for line in f.readlines():
            if json_flag:
                results.append(json.loads(line))
            else:
                results.append(line.strip())
        if all_flag:
            if json_flag:
                return json.loads(''.join(results))
            else:
                return '\n'.join(results)
        return results
    


def get_company_full_names():
    results = []
    for ele in get_datas('data/公司简称检索结果.json'):
        company_name = ele['company']
        search_results = ele['search_results']
        if isinstance(search_results, str):
            results.append(f'{company_name}\t{company_name}')
            continue
        
        prompt = PROMPT.format(company_name=company_name, search_results=search_results)
        result = ''
        while result == '':
            try:
                result = DDGS().chat(prompt, model='gpt-4o-mini')
                if result.strip(): break
            except Exception as e:
                time.sleep(2)
                continue
        results.append(f'{company_name}\t{result}')
    save_datas('data/公司全称提取结果.txt', results, json_flag=False)

if __name__ == '__main__':
    get_company_full_names()
```


View Code
## **5\.总结**


一句话足矣\~


开发了一款基于公司简称补全公司全称的工具，包括具体的框架、实现原理以及完整源码，满满诚意，提供给各位看官。欢迎转发、订阅\~有问题可以私信或留言沟通！


虽然需求比较简单，且实现过程也比较简单，但通过搜索引擎搜素以及大模型的各种奇技淫巧，相信可以完成更加复杂且效果惊艳的项目。


有兴趣的客官可以进行沟通合作，感谢\~


## **6\.参考**


(1\) Duckduckgo\_search: [https://github.com/deedy5/duckduckgo\_search.git](https://github.com)


 ![](https://img2024.cnblogs.com/blog/602535/202411/602535-20241118180503925-394442958.png)


 


 本博客参考[wgetCloud机场](https://tabijibiyori.org)。转载请注明出处！
