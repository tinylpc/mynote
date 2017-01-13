# mynote
a note with my daily

# view的touch分发机制
![](https://github.com/tinylpc/mynote/blob/master/view%E7%9A%84touch%E5%88%86%E5%8F%91.png)

# LayoutInflater参数作用

### 在Android中将一个layout布局转换为view，我们需要用到LayoutInflater，LayoutInflater提供了3种方法实例化一个view，现在主要记录下这3种方法的具体作用。

### 前提参数
主布局，activity_main.xml
```
<?xml version="1.0" encoding="utf-8"?>
<LinearLayout
    xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:tools="http://schemas.android.com/tools"  
    android:id="@+id/activity_main"   
    android:layout_width="match_parent"   
    android:layout_height="match_parent"   
    android:background="#9696ac"
    tools:context="tiny.guavademo.MainActivity">
</LinearLayout>
```

需要实例化的布局，text_test.xml
```
<?xml version="1.0" encoding="utf-8"?>
<TextView  
     xmlns:android="http://schemas.android.com/apk/res/android"    
     android:layout_width="200dp"    
     android:layout_height="100dp"    
     android:background="#220fab"    
     android:orientation="vertical"    
     android:text="hello world"    
     android:textColor="#ffffff"    
     android:textSize="30sp">
</TextView>
```

### 先将inflate源码贴上，便于后续分析
```
1    public View inflate(XmlPullParser parser, @Nullable ViewGroup root) {
2       return inflate(parser, root, root != null);
3    }
```

```
4    public View inflate(@LayoutRes int resource, @Nullable ViewGroup root, boolean attachToRoot) {
5       final Resources res = getContext().getResources();
6       if (DEBUG) {
7           Log.d(TAG, "INFLATING from resource: \"" + res.getResourceName(resource) + "\" ("
8                   + Integer.toHexString(resource) + ")");
9       }

11        final XmlResourceParser parser = res.getLayout(resource);
12        try {
13            return inflate(parser, root, attachToRoot);
14        } finally {
15            parser.close();
16        }
17    }
```

```
18     public View inflate(XmlPullParser parser, @Nullable ViewGroup root, boolean attachToRoot) {
19            synchronized (mConstructorArgs) {
20                Trace.traceBegin(Trace.TRACE_TAG_VIEW, "inflate");

22                final Context inflaterContext = mContext;
23                final AttributeSet attrs = Xml.asAttributeSet(parser);
24                Context lastContext = (Context) mConstructorArgs[0];
25                mConstructorArgs[0] = inflaterContext;
26                View result = root;

28                try {
29                    // Look for the root node.
30                    int type;
31                    while ((type = parser.next()) != XmlPullParser.START_TAG &&
32                            type != XmlPullParser.END_DOCUMENT) {
33                        // Empty
34                    }

35                    if (type != XmlPullParser.START_TAG) {
36                        throw new InflateException(parser.getPositionDescription()
37                                + ": No start tag found!");
38                    }

40                    final String name = parser.getName();
                
42                    if (DEBUG) {
43                        System.out.println("**************************");
44                        System.out.println("Creating root view: "
45                                + name);
46                        System.out.println("**************************");
47                    }

49                    if (TAG_MERGE.equals(name)) {
50                        if (root == null || !attachToRoot) {
51                            throw new InflateException("<merge /> can be used only with a valid "
52                                    + "ViewGroup root and attachToRoot=true");
53                        }

55                        rInflate(parser, root, inflaterContext, attrs, false);
56                    } else {
57                        // Temp is the root view that was found in the xml
58                        final View temp = createViewFromTag(root, name, inflaterContext, attrs);

60                        ViewGroup.LayoutParams params = null;

62                        if (root != null) {
63                            if (DEBUG) {
64                                System.out.println("Creating params from root: " +
65                                        root);
66                            }
67                            // Create layout params that match root, if supplied
68                            params = root.generateLayoutParams(attrs);
69                            if (!attachToRoot) {
70                                // Set the layout params for temp if we are not
71                                // attaching. (If we are, we use addView, below)
72                                temp.setLayoutParams(params);
73                            }
74                        }

76                        if (DEBUG) {
77                            System.out.println("-----> start inflating children");
78                        }

79                        // Inflate all children under temp against its context.
80                        rInflateChildren(parser, temp, attrs, true);

81                        if (DEBUG) {
82                            System.out.println("-----> done inflating children");
83                        }

85                        // We are supposed to attach all the views we found (int temp)
86                        // to root. Do that now.
87                        if (root != null && attachToRoot) {
88                            root.addView(temp, params);
89                        }

91                      // Decide whether to return the root that was passed in or the
92                        // top view found in xml.
93                        if (root == null || !attachToRoot) {
94                            result = temp;
95                        }
96                    }

98                } catch (XmlPullParserException e) {
99                    final InflateException ie = new InflateException(e.getMessage(), e);
100                   ie.setStackTrace(EMPTY_STACK_TRACE);
101                   throw ie;
102               } catch (Exception e) {
103                   final InflateException ie = new InflateException(parser.getPositionDescription()
104                           + ": " + e.getMessage(), e);
105                   ie.setStackTrace(EMPTY_STACK_TRACE);
106                   throw ie;
107               } finally {
108                   // Don't retain static reference on context.
109                   mConstructorArgs[0] = lastContext;
110                   mConstructorArgs[1] = null;

112                   Trace.traceEnd(Trace.TRACE_TAG_VIEW);
113               }

115               return result;
116           }
117       }
```

### 第一种方式
```
setContentView(R.layout.activity_main);
LinearLayout ll = (LinearLayout)findViewById(R.id.activity_main);
View view = LayoutInflater.from(this).inflate(R.layout.text_test,null);
ll.addView(view);
```

### 效果图

![](https://github.com/tinylpc/mynote/blob/master/%E6%95%88%E6%9E%9C1.png)

TextView设置的宽高完全没效果，为什么呢，开始分析源码。
因为传入的root值为null,所以attachToRoot为false，见第2行源码。然后会执行到第58行，通过createViewFromTag获取一个view实例，这个实例的LayoutParams为null，再看第94行，这个view会作为最终结果返回。下面就是LinearLayout的addView方法。
```

LayoutParams params = child.getLayoutParams();
if (params == null) {
params = generateDefaultLayoutParams();
    if (params == null) {
        throw new IllegalArgumentException("generateDefaultLayoutParams() cannot return null");
    }
}
        
protected LayoutParams generateDefaultLayoutParams() {
   return new LayoutParams(LayoutParams.WRAP_CONTENT, LayoutParams.WRAP_CONTENT);
}
```
因为view的LayoutParams为null，这时会创建默认的LayoutParams，都是WRAP_CONTENT，所以无论怎么修改TextView的宽高属性都不会生效

### 第二种方式
```
setContentView(R.layout.activity_main);
LinearLayout ll = (LinearLayout)findViewById(R.id.activity_main);
View view = LayoutInflater.from(this).inflate(R.layout.text_test,ll,true);
```

### 效果图
![]()
