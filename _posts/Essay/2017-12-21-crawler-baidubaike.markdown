---
layout:     post
title:      "WebMagic实现百度百科爬虫"
subtitle:   "Baidu Baike Crawler"
date:       2017-12-21
author:     "Jon Lee"
header-img: "img/in-post/2017-12-21-crawler-baidubaike/bg.jpg"
catalog:    true
categories: 随笔
tags:
    - Java
    - 爬虫
---

### 爬取目标

最近需要对一些领域概念做分析，选择利用百度百科爬取对应的词条，从中获取信息和知识。  
我将查询结果分为两类，一种是百科中已收录，另一种就是未被收录，虽然未被收录但还是能从中获取一些信息。  
对于未被收录的词条，会给出与其相关的若干词条，所以这一部分也作为爬取内容，可能会提取到有价值的信息。

![](/img/in-post/2017-12-21-crawler-baidubaike/1.jpg)

而对于已收录的词条，我需要的信息是 **标题**、**同义词**、**摘要**、**Infobox**、**百度知心**。

![](/img/in-post/2017-12-21-crawler-baidubaike/2.jpg)

### 主要代码

    public class BaiduBaikeSpider implements PageProcessor {

    	@Override
    	public void process(Page page) {
    		if (page.getUrl().regex("https://baike.baidu.com/item/.*").match()) {

    			Html html = page.getHtml();

    			if (html.xpath("/html/body/div[3]/div[2]/div/div[2]/dl[1]/dd/h1/text()").match()) {
    				// 标题
    				String title = html.xpath("/html/body/div[3]/div[2]/div/div[2]/dl[1]/dd/h1/text()").toString();
    				// 同义词
    				String synonym = html.xpath("/html/body/div[3]/div[2]/div/div[2]/span/span/text()").all().toString();
    				// 摘要
    				String summary = html.xpath("/html/body/div[3]/div[2]/div/div[2]/div[@class='lemma-summary']/allText()")
    						.all().toString().replaceAll(",", "");
    				// 信息框
    				String infobox = html
    						.xpath("/html/body/div[3]/div[2]/div/div[2]/div[@class='basic-info cmn-clearfix']/dl/allText()")
    						.all().toString();

    				StringBuilder sb = new StringBuilder();
    				sb.append(word + "\t" + title + "\t" + synonym + "\t" + summary + "\t" + infobox + "\n");
    				try {
    					outPut(PATH + "百科词条.txt", sb.toString());
    				} catch (IOException e) {
    					e.printStackTrace();
    				}

    				// 百度知心,由于是动态加载，先获取id
    				String zhixinID = html.xpath("//*[@id=\"zhixinWrap\"]/@data-newlemmaid").toString();
    				String zhixinURL = "https://baike.baidu.com/wikiui/api/zhixinmap?lemmaId=" + zhixinID;
    				// 添加任务
    				page.addTargetRequest(zhixinURL);

    			}
    			// 如果不存在词条则获取最相关的词条，添加url到任务
    			else {
    				page.addTargetRequest("https://baike.baidu.com/search/none?word=" + word);
    			}
    		}
    		// 解析百度知心Json
    		else if (page.getUrl().regex("https://baike.baidu.com/wikiui/.*").match()) {
    			Map<String, List<String>> resultMap = new LinkedHashMap<>();
    			// 获取相关类型的数量，因为有的词是两个，有些是三个
    			List<String> tempList = new JsonPathSelector("$.[*]").selectList(page.getRawText());
    			int typeNums = tempList.size();
    			int count = typeNums;
    			while (count > 0) {
    				resultMap.put(new JsonPathSelector("$.[" + (typeNums - count) + "].tipTitle").select(page.getRawText()),
    						new JsonPathSelector("$.[" + (typeNums - count) + "].data[*].title")
    								.selectList(page.getRawText()));
    				count--;
    			}
    			StringBuilder sb = new StringBuilder();
    			sb.append(word + "\t");
    			resultMap.forEach((key, value) -> {
    				sb.append(key + "：" + value + "\t");
    			});
    			sb.append("\n");
    			try {
    				outPut(PATH + "相关词条_知心.txt", sb.toString());
    			} catch (IOException e) {
    				e.printStackTrace();
    			}
    		}
    		// 百度百科尚未收录词条，获取相关词条
    		else if (page.getUrl().regex("https://baike.baidu.com/search/none\\?word=.*").match()) {
    			List<String> list = page.getHtml().xpath("//*[@id=\"body_wrapper\"]/div[1]/dl/dd/a/@href").all();
    			try {
    				list = UrlDecode(list);
    			} catch (UnsupportedEncodingException e) {
    				e.printStackTrace();
    			}
    			StringBuilder sb = new StringBuilder();
    			sb.append(word + "\t" + list + "\n");
    			try {
    				outPut(PATH + "相关词条_未收录.txt", sb.toString());
    			} catch (IOException e) {
    				// TODO Auto-generated catch block
    				e.printStackTrace();
    			}
    		} else
    			System.out.println("Nothing!");
    	}
    	// URL解码
    	private static List<String> UrlDecode(List<String> rawList) throws UnsupportedEncodingException {
    		List<String> resultList = new LinkedList<>();
    		String reg = "https://baike.baidu.com/item/(.*)/\\d+";
    		Pattern p = Pattern.compile(reg);
    		Matcher m;
    		for (String str : rawList) {
    			m = p.matcher(str);
    			if (m.find()) {
    				resultList.add(java.net.URLDecoder.decode(m.group(1), "UTF-8"));
    			}
    		}
    		return resultList;
    	}
    }

### 结果截图

**词条的基本信息**：

![](/img/in-post/2017-12-21-crawler-baidubaike/3.jpg)

**Infobox**：

![](/img/in-post/2017-12-21-crawler-baidubaike/3-2.jpg)

**百度之心内容**：
![](/img/in-post/2017-12-21-crawler-baidubaike/4.jpg)

---
