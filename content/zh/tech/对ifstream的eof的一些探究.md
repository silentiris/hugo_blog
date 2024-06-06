+++
title = "对ifstream的eof的一些探究"
date = "2024-06-06T11:23:15+08:00"
tags = []
slug = "some-explorations-of-the-eof-of-ifstream"
+++

在写数据库比赛的时候遇到了这样一个问题：我需要从一个二进制文件中读取先前存入的数数据，基本的读取代码长这样

```c++
std::vector<std::ifstream> inter_files(file_cnt);
......
if (!inter_files[file_idx].eof()) {
        auto rec_tmp = std::make_unique<RmRecord>(executor->tupleLen());
        for (const auto &col : executor->cols()) {
          inter_files[file_idx].read(rec_tmp->data + col.offset, col.len);
        }
        LOG_DEBUG("read_size:"+std::to_string(inter_files[file_idx].gcount()));
        min_heap.emplace(std::move(rec_tmp), file_idx);
      }
```

大眼一看其实是没什么问题的，但是测试之后却发现，每次for循环会读8个字节，原始文件应该只有24个字节，所以一共只应该循环三次，但是最后却循环了四次。直接反映到的就是输出文件会多出一行0。

![image-20240606111235227](https://cdn.jsdelivr.net/gh/silentiris/pic_bed@latest/blog-images/image-20240606111235227.png)

排查了一番后发现问题出在`std::ifstream`的`eof()`方法上了。eof方法的原理是监测你最后读取的是否是文件结束符，文件结束符并不是文件的最后一个字符，而是最后一个字符的下一个字符0xFF，所以在读取24字节后，eof判断是我刚刚读入的四个字节是不是文件结束符，当然不是，所以就会再次进入循环，直到下次循环调用`read()`后读取到了0xFF，再次判断`eof()`的时候才会认为读取到了eof，才会终止运行。了解了问题产生的原因，相应更改代码逻辑来处理即可。