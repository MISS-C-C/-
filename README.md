# -
爬虫
from selenium import webdriver
from selenium.common.exceptions import TimeoutException
from selenium.webdriver.common.by import By
from selenium.webdriver.support import expected_conditions as EC
from selenium.webdriver.support.wait import WebDriverWait
from urllib.parse import quote
from pyquery import PyQuery as pq
import pymongo

MAX_PAGE = 100
MONGO_URL = 'localhost'
MONGO_DB = 'taobao'
MONGO_COLLECTION = 'foods'
client = pymongo.MongoClient(MONGO_URL)
db = client[MONGO_DB]

browser = webdriver.Chrome()
wait = WebDriverWait(browser, 10)
KEYWORD = '美食'

def index_page(page):
"""
抓取索引页：param page：页码
"""
print('正在爬取第', page, '页')
try:
url = 'https://s.taobao.com/search?q=' + quote(KEYWORD)
browser.get(url)
if page > 1:
input = wait.until(
EC.presence_of_element_located((By.CSS_SELECTOR, '#mainsrp-pager div.form > input')))
submit = wait.until(
EC.element_to_be_clickable((By.CSS_SELECTOR, '#mainsrp-pager div.form > span.btn.J_Submit')))
input.clear()
input.send_keys(page)
submit.click()
wait.until(
EC.text_to_be_present_in_element((By.CSS_SELECTOR, '#mainsrp-pager li.item.active > span'), str(page)))
wait.until(EC.presence_of_element_located((By.CSS_SELECTOR, '.m-itemlist .items .item')))
get_products()
except TimeoutException:
index_page(page)

def get_products():
'''
提取商品
'''
html = browser.page_source
doc = pq(html)
items = doc('#mainsrp-itemlist .items .item').items()
for item in items:
product = {
'image': item.find('.pic .img').attr('data-src'),
'price': item.find('.price').text(),
'deal': item.find('.deal-cnt').text(),
'title': item.find('.title').text(),
'shop': item.find('.shop').text(),
'location': item.find('.location').text()
}
print(product)
save_to_mongo(product)

def main():
'''
遍历每一页
'''
for i in range(1, MAX_PAGE+1):
index_page(i)
browser.close()

def save_to_mongo(result):
"""
保存至MongoDB
"""
try:
if db[MONGO_COLLECTION].insert(result):
print('存储到MongoDB 成功')
except Exception:  
print('存储到MongoDB失败')

if __name__ == '__main__':
main()


------后续数据清洗

import pandas as pd
import numpy as np
import pymysql
import re
coon = pymysql.connect(
    host='localhost', user='root', passwd='root',
    port=3306, db='taobao', charset='utf8'
    # port必须写int类型
    # charset必须写utf8，不能写utf-8
)
cur = coon.cursor()  # 建立游标
sql='select * from taobao_food'
df=pd.read_sql(sql=sql,con=coon)
#print(df.values)
df=pd.DataFrame(df)
df=df.drop('id',axis=1)
print(pd.isnull(df).values.any())

2、去重

print('去重之前的形状',df.shape)
df=df.drop_duplicates(keep='first')
print('去重之后的形状',df.shape)
print(df.head())


3、提取地址信息以及购买数量

def get_buy_num(buy_num):
    if u'万' in buy_num:  # 针对1-2万/月或者10-20万/年的情况，包含-
        buy_num=float(buy_num.replace("万",''))*10000
        #print(buy_num)
    else:
        buy_num=float(buy_num)
    return buy_num
df['place'] = df['place'].replace('','未知')#fillna("['未知']")
datasets = pd.DataFrame()
for index, row in df.iterrows():
    #print(row["place"])
    row["place"] = row["place"][:2]
    row["buy_num"]=get_buy_num(row["buy_num"][:-3].replace('+',''))
    #print(row["place"])
df.to_csv('taobao_food.csv',encoding='utf8',index_label=False)
