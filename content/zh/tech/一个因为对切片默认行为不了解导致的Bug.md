+++
title = "一个因为对切片默认行为不了解导致的Bug"
date = "2024-03-17T22:32:44+08:00"
tags = []
slug = "A-bug-caused-by-ignorance-of-the-default-behavior-of-slicing"
+++
```go
titles := make([]string, len(acSubmissions))
	for _, acSubmission := range acSubmissions {
		titles = append(titles, acSubmission.Question.Title)
		hlog.Info(acSubmission)
	}
	hlog.Info(titles)
	global.RDB.SAdd("leetcode::history-ac::"+req.Username, titles)
```

这一段平白无奇的代码执行之后发现总是会莫名其妙的多出一个空的

```bash
127.0.0.1:6379> SMEMBERS leetcode::history-ac::silentdragon
 1) "First Missing Positive"
 2) "Trapping Rain Water"
 3) "Valid Sudoku"
 4) ""
 5) "Count and Say"
 6) "Search in Rotated Sorted Array"
 7) "Two Sum"
 8) "Search Insert Position"
 9) "Find First and Last Position of Element in Sorted Array"
10) "Combination Sum"
11) "Combination Sum II"
```

最后发现原因是因为在上面make创建切片后给前len个都分配一个空字符串，append是从第len开始往后append的，所有会有空串的出现

```go
	strs := make([]string, 10)
	strs = append(strs, "hello")
	fmt.Println(len(strs))
```

类比这段代码，会输出11，这就是这个问题的所在。搞混了java和go去创建一个slice/list类型的区别，以为go的make也是像java一样预留空间，防止频繁resize。
