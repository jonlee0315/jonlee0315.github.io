---
layout:     post
title:      "Trie的实现及过滤敏感词"
subtitle:   ""
date:       2018-08-01
author:     "Jon Lee"
header-img: "img/in-post/algorithm-and-data-structure/trie/bg.jpg"
catalog: true
categories : 算法与数据结构
tags:
    - 算法
    - 查找
    - Tree
    - Java
---

### 概述

在计算机科学中，trie，又称前缀树或字典树，是一种有序树，用于保存关联数组，其中的键通常是字符串。与二叉查找树不同，键不是直接保存在节点中，而是由节点在树中的位置决定。一个节点的所有子孙都有相同的前缀，也就是这个节点对应的字符串，而根节点对应空字符串。一般情况下，不是所有的节点都有对应的值，只有叶子节点和部分内部节点所对应的键才有相关的值。

### 实现

    public class TrieNode {
        private boolean end = false;

        private Map<Character, TrieNode> subNodes = new HashMap<>();

        void addSubNode(Character key, TrieNode node) {
            subNodes.put(key, node);
        }

        TrieNode getSubNode(Character key) {
            return subNodes.get(key);
        }

        boolean isKeywordEnd() {
            return end;
        }

        void setKeywordEnd(boolean end) {
            this.end = end;
        }
    }

添加敏感词

    private void addWord(String lineTxt) {
        TrieNode tempNode = rootNode;
        for (int i = 0; i < lineTxt.length(); ++i) {
            Character c = lineTxt.charAt(i);
            if (isSymbol(c)) {
                continue;
            }
            TrieNode node = tempNode.getSubNode(c);
            if (node == null) { // 没初始化
                node = new TrieNode();
                tempNode.addSubNode(c, node);
            }
            tempNode = node;
            if (i == lineTxt.length() - 1) {
                tempNode.setKeywordEnd(true);
            }
        }
    }

判断字符是否为符号

    private boolean isSymbol(char c) {
        int ic = (int) c;
        // 0x2E80-0x9FFF 东亚文字范围
        return !Character.isAlphabetic(c) && (ic < 0x2E80 || ic > 0x9FFF);
    }

过滤函数

    public String filter(String text) {
		if (StringUtils.isEmpty(text)) {
			return text;
		}
		String replacement = DEFAULT_REPLACEMENT;
		StringBuilder result = new StringBuilder();

		TrieNode tempNode = rootNode;
		int begin = 0; // 回滚数
		int position = 0; // 当前比较的位置

		while (begin < text.length()) {
			if (position == text.length()) {
				position = begin;
				continue;
			}
			char c = text.charAt(position);
			// 符号跳过
			if (isSymbol(c)) {
				if (tempNode == rootNode) {
					result.append(c);
					++begin;
				}
				position++;
				continue;
			}
			tempNode = tempNode.getSubNode(c);
			if (tempNode == null) {
				result.append(text.charAt(begin));
				position = ++begin;
			} else if (tempNode.isKeywordEnd()) {
				result.append(replacement);
				begin = ++position;
			} else {
				if (++position == text.length()) {
					result.append(text.charAt(begin));
					position = ++begin;
				} else {
					continue;
				}
			}
			tempNode = rootNode;
		}
		// result.append(text.substring(begin));
		return result.toString();
	}
