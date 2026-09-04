---
title: "C语言词法分析器 Lexer"
date: 2026-04-07T20:24:33+08:00
description: "C 语言词法分析器 Lexer 的 Token 定义与实现示例。"
categories: ["项目笔记"]
tags: ["Lexer", "词法分析", "C语言"]
---

C 语言词法分析器 Lexer。

翻译：

```c
int a = 10;
```

## Token 类型定义

```c
//.h

typedef enum{
    TOKEN_KEYWORD_INT,            //关键词-INT
    TOKEN_IDENTIFIER_VARIABLE,    //标识符-变量
    TOKEN_ASSIGN,                 //赋值
    TOKEN_NUMBER,                 //数字
    TOKEN_SEMICOLON,              //分号
    TOKEN_EOF,					//结束符
}TOKEN_TYPE;

typedef struct{
    TOKEN_TYPE ;
    char str[120];
}TokenData;

extern TokenData tokenData;
```

## lexer 功能示例

```c
/* lexer功能示例
输入： int a = 10;
输出 INT, IDENTIFIER(a), ASSIGN, NUMBER(10), SEMICOLON
*/
#include <stdio.h>
#include <string.h>
#include <ctype.h>
TokenData tokenData;

void lexer_run(FILE *f)
{
    int i = 0;
    
	//跳过空格
    while((c = fgetc(f)) != EOF && isspace(c));
    
    //读到EOF
    if(c == EOF)
    {
        tokenData.token_type = TOKEN_EOF;
        return;
    }
    
    //读到字母
    if(isalpha(c))
    {
        i = 0;
        tokenData.str[i++] = c;
        while(isalnum(c = fgetc(f))  //字母和数字接着读
        {
            tokenData.str[i++] = c;
        }
        tokenData.str[i] = 0;     
        ungetc(c, f);
        if(strcmp(tokenData.str, "int"))
		{
         	 tokenData.token_type = TOKEN_IDENTIFIER_VARIABLE;	        
  		}
  		else
  		{
  	          tokenData.token_type = TOKEN_KEYWORD_INT;  
 	 	}
         return;
    }
    
 	//读到数字
     if(isdigit(c))
	{
        i = 0;
        tokenData.str[i++] = c;
        while(isdigit(c = fgetc(f))  //数字接着读
        {
            tokenData.str[i++] = c;
        }
        tokenData.str[i] = 0;     
        ungetc(c, f);
		tokenData.token_type = TOKEN_NUMBER;	        
         return;
	}
              
    switch(c)
  	{
        case '=':
        	memcpy(tokenData.str, "=", 1);
        	tokenData.token_type = TOKEN_ASSIGN;
        	break;
        case ';':
            memcpy(tokenData.str, ";", 1);
        	tokenData.token_type = TOKEN_SEMICOLON;
        	break;
        default:
        	break;
  	}
}
```
