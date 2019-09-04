---
title: "Looking at My Programming Using File Extensions "
summary: Use file extensions to see how much I have programmed in various languages
date: 2019-08-29
authors: ["admin"]
tags: ["python", "analysis"]
output: html_document
---



## Introduction

I was recently fooling around with listing filenames in a specific folder in python, when I realized I could use that to count up all the file extensions in a folder and its subfolders. Since code files always have specific extensions, I decided to use this to look at what my production is like in different programming langauges and see what I could learn from the data.

## Collecting the Data

The only languages I program in enough to warrant analyzing are R, Python, and C++, so I just needed to look for text files and notebooks designated for those languages. I have kept most of the programming I have done since starting college in 2015, so I will scan through all the files in my drive.



```python
import seaborn as sns
import pandas as pd
import os, re
import numpy as np
from json import load
from sys import argv
import matplotlib.pyplot as plt
```




```python
mypath = "/Users/kevinmacdonald/Google Drive/"
code_files = []
code_ext = []
code_type = []
file_extensions = ["\.R$", "\.py$", "\.cpp$", "\.ipynb$", "\.Rmd$"]


# Get file paths
for path, subdirs, files in os.walk(mypath):
    for name in files:
        for extension in file_extensions:
            if re.search(extension, name):
                code_files.append(os.path.join(path, name))
                code_ext.append(extension)
                if "py" in extension:
                  code_type.append("Python")
                elif "R" in extension:
                  code_type.append("R")
                else:
                  code_type.append("C++")


line_lengths = np.zeros(len(code_files))
line_counts = {ext:0 for ext in file_extensions}
error_counts = {ext:0 for ext in file_extensions}

# Collect information
for i, file in enumerate(code_files):
    # file_len is custom function
    line_count, error = file_len(file, code_ext[i])
    line_lengths[i] = line_count
    line_counts[code_ext[i]] += line_count
    error_counts[code_ext[i]] += error

sizes = pd.DataFrame({"file":code_files, 
                      "length":line_lengths, 
                      "ext":code_ext,
                      "lang":code_type})
sizes['path'] = sizes['file'].str.split("/").apply(lambda x: x[1:-1])
sizes['file'] = sizes['file'].str.split("/").apply(lambda x: x[-1])
```


```python
sizes[['file','length','ext', 'lang']].head()
```

```
##               file  length     ext    lang
## 8   academic.ipynb     3.0  .ipynb  Python
## 9     sync_i18n.py    52.0     .py  Python
## 10      kd_tree.py    64.0     .py  Python
## 11         test.py    14.0     .py  Python
## 12  plotting.ipynb    22.0  .ipynb  Python
```
The data frame contains the name of each file, the path, the number of lines in the file, and what type of file it is. I can now look at how many files of each type I have.

<img src="/www.kevinmacdonald.me/post/code_analysis_files/figure-html/bar_plot-1.png" width="768" style="display: block; margin: auto;" />

As I would have expected, I have the most R files out of the three langauges. R is the language I use most frequently and have been using it for the past 4 years. I have only really started using Python for the past 8 months while C++ was actually the first language I learned as a freshman in college. 

<img src="/www.kevinmacdonald.me/post/code_analysis_files/figure-html/box_plot-1.png" width="768" style="display: block; margin: auto;" />

This data also allows me to compare the lengths of files by langauges. I limited the data to files under 300 lines just to make the visualization easier to read. Interestingly, median length of files is relatively the same across languages; I assummed C++ files would be much longer than the rest, but there isn't that much of a difference, just about 5 lines longer. R files, however, are heavily skewed with many outliers. This is because most of my college classes  used R, so I have lots of final projects written in R. 

Now that I've looked at these files in aggregate, I'm interested to see what my longest file is for each language.


```python
# analyze files for each language

large_files = []
large_size = []
languages = ["R", "Python", "C++"]
for lang in languages:
    large_files.append(sizes['file'][sizes[sizes['lang'] == lang]['length'].idxmax()].split("/")[-1])
    large_size.append(sizes[sizes['lang'] == lang]['length'].max())
    
largest = pd.DataFrame({"file":large_files,
                         "size":large_size})
largest.head()
```

```
##                        file   size
## 0    Final_Presentation.Rmd  333.0
## 1  website_clustering.ipynb  196.0
## 2                  hand.cpp  493.0
```

For each language, my longest file created is from a final project for a class. The R file is from a project comparing combinatorial optimization algorithms, the Python from a project building a website recommendation system, and the C++ one from a project building a command line poker game to play vs a computer. 

```python
sizes.groupby("lang").agg({"length":"sum"})
```

```
##          length
## lang           
## C++     11063.0
## Python   2851.0
## R       15704.0
```

Overall, I have written over 10,000 lines of code for both R and C++, although since C++ code is much more verbose than R or Python, that is inflated some. Python is something I am still learning so I have only written about 3000 lines of code there. However, these totals do not include all the deletions, modifications, dead ends, etc. that I have run into over the years I have been programming which would probably be 10x as large as the actual finished code I've produced.
