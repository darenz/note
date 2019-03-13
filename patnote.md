* getline(cin,string)
* char s[100]; cin.getline(s,100);
* cin 与 getline 连续使用时，getline会吃掉上一个cin的回车
    > cin.ignore() ; cin.get() ; 跳过字符,不能用getchar(),猜测getchar默认跳过制表符，需要使用流处理函数
* cin   cin.get()   cin.getline()   getline()   gets()  getchar()


* sprintf(s,"%d",n);
    在数字转字符串时非常有用

* 数字黑洞，输入6174本身的例子

* long 输出格式： ld ;  
* long long 输出格式： lld;

* hash表长度最好为质数。注意处理表长为1

* 在下一次循环中处理或判断的情况，要注意末尾多一次循环

* 细心，注意返回的true,false不要写反，使用set/map/pair注意用的第一个还是第二个元素。（1116,C比赛题，小黄人）

* 1117爱丁顿数。理解题意有问题，神也救不了

* bool返回的函数，注意true都要显式返回。否则默认返回为false