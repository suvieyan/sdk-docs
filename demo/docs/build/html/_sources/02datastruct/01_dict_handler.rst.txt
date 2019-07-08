一个有趣的字典排序问题
=========================

需求：字典key升序, 大小写不敏感，value升序，大小写敏感
------------------------------------------------------

::

    dic = {
    'name': 'yan',
    'NAME': 'zhang',
    'naMe': 'Zhang',
    'age': 'yan',
    'NAme': 'xi',
    'Name': 'wang',
    'NamE': 'alex',
    }

    ret = sorted(dic.items(), key=lambda val: (val[0].lower(), val[1]))
    print(ret)
    ## 结果：
    [('age', 'yan'), ('naMe', 'Zhang'), ('NamE', 'alex'), ('Name', 'wang'), ('NAme', 'xi'), ('name', 'yan'), ('NAME', 'zhang')]