# 使用ANNOVAR注释人群变异

作者 · 不加酵母的面团（DoughWithoutYeast）

原发表于 2026-09-16 · [博客园原文](https://www.cnblogs.com/DoughWithoutYeast/p/22993924)

原文更新于 2026-10-26

## 1\. 介绍ANNOVAR

![image](3805523-20260916111149252-335647969.png)

ANNOVAR就像是它字面的意思ANN(otate) VAR(iants)一般，主要是对变异位点进行注释的软件。`用于注释单核苷酸多态性（SNV）以及插入/缺失突变，例如分析其对基因的功能影响、推断染色体带区、报告功能重要性评分、在保守区域中发现变异，或识别来自1000基因组计划和dbSNP数据库中的变异`。官方介绍。

## 2\. 下载ANNOVAR

<https://annovar.openbioinformatics.org/en/latest/> ANNOVAR官网！非常非常详尽的帮助文档！(如果分析中遇到问题，去github内找找看有没有类似的issue，或者直接提问，作者非常活跃解答的！)

首先，点击`quick starup guide`，在`Download and install`进入下载页面。输入**学校邮箱**，就可以等待下载的安装包啦！我当时赶时间，用的是师兄先前下载好的安装包直接进行安装的！

PS：还是建议大家都创建一个学校邮箱哦\~ 感觉这个算是一个学术的认证感觉了！会给人比较可靠的感觉嘿嘿嘿！

随后，根据教程进行解压缩，就可以使用annovar啦\~

ANNOVAR软件主要包括下面的脚本：

```
annotate_variation.pl # 可以用来下载数据库
coding_change.pl
convert2annovar.pl # 构建vcf文件的脚本
example
humandb # 下载的数据库存放的目录
retrieve_seq_from_fasta.pl
table_annovar.pl # 注释脚本
variants_reduction.pl
```

至此，我们就可以着手分析啦\~

## 3\. 下载ANNOVAR数据库信息

在正式分析之前，我们需要做一些准备工作。因为ANNOVAR其实是将变异注释到对应的数据库内，ANNOVAR自带了一些数据库，但是有的时候我们需要个性化注释一些内容时就需要提前下载相关的数据库！

<https://annovar.openbioinformatics.org/en/latest/user-guide/download/> 基本涵盖了ANNOVAR可以注释的数据库的内容。<strong>这里我们需要选择与变异位点匹配的版本的数据库信息！</strong>文档内有注释各个数据库的内容！

**这里很多时候数据库的名称可能有所变更**

分享一下我注释的数据库的下载代码：

```bash
perl /path/to/annovar/annotate_variation.pl -buildver hg38 -downdb -webfrom annovar refGene  humandb/
perl /path/to/annovar/annotate_variation.pl -buildver hg38 -downdb cytoBand  humandb/
perl /path/to/annovar/annotate_variation.pl -buildver hg38 -downdb -webfrom annovar dbnsfp47a  humandb/
perl /path/to/annovar/annotate_variation.pl -buildver hg38 -downdb -webfrom annovar avsnp151  humandb/
perl /path/to/annovar/annotate_variation.pl -buildver hg38 -downdb -webfrom annovar gnomad211_exome  humandb/
perl /path/to/annovar/annotate_variation.pl -buildver hg38 -downdb -webfrom annovar gnomad_exome humandb/
perl /path/to/annovar/annotate_variation.pl -buildver hg38 -downdb -webfrom annovar revel humandb/
perl /path/to/annovar/annotate_variation.pl -buildver hg38 -downdb -webfrom annovar gnomad211_genome humandb/
```

**这里`-buildver hg38`即指示下载的数据库对应的基因组的版本，要与即将分析的变异位点的版本相对应！**

## 4\. 使用annovar注释变异

现在可以对我们的变异位点进行处理啦\~

### 4.1 构造input文件

首先，我们需要构建合格的可以用于ANNOVAR分析的文件。\
ANNOVAR支持的文件类型有：

* VCF
* avinput(可以有VCF文件转换)

这里我介绍主要接受使用avinput作为输入的方法。

使用内置的`convert2annovar.pl`脚本就可以将VCF文件转成avinput格式。

```bash
#!/bin/bash
vcf_dir="/path/to/vcf/" # 放所有vcf文件的目录
output_dir="/path/to/avinput/"

for vcf_file in $vcf_dir/*.vcf; do
  base_name=$(basename "$vcf_file")
  output_prefix="${base_name%.*}"
  perl /path/to/annovar/convert2annovar.pl -format vcf4 "$vcf_file" -out "$output_dir/$output_prefix.avinput"
done
echo "Finished processing all VCF files."
```

将上方的脚本中：`vcf_dir`和`output_dir`两个变量修改成你自己的vcf文件以及想要的avinput文件的存放路径，并保存成`.sh`文件，使用`bash`运行，就可以遍历`vcf_dir`内的所有VCF文件，将他们各自转成avinput文件，并保存在`avinput`目录下面

### 4.2 true 注释

接下来就可以进行真正的注释啦

```bash
#!/bin/bash
annovar_vcf_dir="/path/to/avinput/" # 前面转成的avinput文件的路径
output_dir="/path/to/csv_annotated/" # 你想要的结果csv文件的存放路径
if [ ! -d "$output_dir" ]; then
  mkdir -p "$output_dir"
fi
for avinput_file in $annovar_vcf_dir/*.avinput; do
  if [ -f "$avinput_file" ]; then
    base_name=$(basename "$avinput_file")
    output_prefix="${base_name%.*}"
    output_full_path="$output_dir/$output_prefix"
    perl /path/to/annovar/table_annovar.pl "$avinput_file" /path/to/annovar/humandb/ \
      -buildver hg38 \ # 注意这里版本要和下载的数据库保持一致
      -out "$output_full_path" \
      -remove \
      -protocol refGene,cytoBand,exac03,avsnp147,dbnsfp30a,1000g2015aug_all,1000g2015aug_afr,1000g2015aug_eas,1000g2015aug_eur,gnomad_exome,gnomad_genome,gnomad211_exome,gnomad211_genome,revel,gerp++gt2 \
      -operation g,r,f,f,f,f,f,f,f,f,f,f,f,f,f \
      -nastring . \
      -csvout \
      -polish \
      -xref /path/to/annovar/example/gene_xref.txt
    
    echo "Processed file: $avinput_file -> $output_full_path"
  else
    echo "File not found: $avinput_file"
  fi
done
echo "Finished processing all .avinput files."
```

**这里有几个注意点：**

* `-buildver` 这里需要和之前下载的数据库的版本对应。如果你只下载了hg19的但是这里选择了hg38的话会报错，但是如果hg19/hg38都下载了，指定了hg19的话，会调用指定版本的数据库！更重要的是要和输入的variants的版本对应上，不同的版本的区别还是很大的！
* `-operation`的参数主要是告诉ANNOVAR对每个数据库是进行如何的操作
  + `-g`表示基于基因的操作
  + `-gx`表示基于基因并带有交叉参考注释（来自-xref参数）的操作
  + `-r`表示基于区域的操作
  + `-f`表示基于过滤的操作
  + 具体的参数可以参考数据库下载条目下的说明

将上面的脚本内的对应路径进行修改，并按照自己的实际情况修改对应的数据库、参考基因组版本等等信息，并保存`.sh`文件后使用`bash`运行即可。

## 5\. 结果怎么看

ANNOVAR的输出结果是以csv文件存放的。其中内容包括：\
![image](3805523-20260916152343679-158454207.png)\
主要内容可以分成两部分：

* 变异基本信息
* ANNOVAR注释的变异在各个数据库(你选定的)的信息(按你设置的顺序排列)

对ANNOVAR的结果我们一般进行一定的筛选，例如致病性/频率等等。经过类似的筛选之后，我们预期找到某一个具有致病性的低频突变。一般来说高频突变的致病性不强，低频突变的致病性更强，且这样的低频突变一般改变那些essential gene的功能，否则这样的个体将在生长过程中被筛选掉。

因此，ANNOVAR注释一般是用于对特定的疾病人群的基因组测序/外显子组测序所找到的变异位点进行注释，并试图找到那些存在特定人群的突变，对这些突变进行注释之后探索他们是如何通过变异导致疾病的，进而对疾病的治疗带来一定的启发！

---

[合集目录](README.md)
