
当父view的mode为EXACTLY的时候，他的宽高就是自己的宽高
如果它的mode是AT_MOST的时候，表示是最大宽度，他的宽高由子view决定

MeasureSpec 有getMode和getSize方法。
32位 高二位表示的是mode，低30位表示的是size

通过子view和父view的MeasureSpec确定mode。

 若父View是EXACTLY，则父View有确切数值或者march_parent
    case MeasureSpec.EXACTLY:
    		若子View的childDimension大于0，则表示有确切数值，则子View大小为其本身且mode是EXACTLY
    		 若子View的childDimension是MATCH_PARENT，则子View的大小为父View的大小且mode是EXACTLY
    		 若子View的childDimension是WRAP_CONTENT，则子View的大小为父View的大小且mode是AT_MOST，表示最大不可超过父View数值

 若父View是AT_MOST，则父View一般是wrap_content，强给子View最大的值
    case MeasureSpec.AT_MOST:
    		若子View的childDimension大于0，则表示有确切数值，则子View大小为其本身且mode是EXACTLY
    		 若子View的childDimension是MATCH_PARENT，则子View的大小不超过父View的大小且mode是AT_MOST
    		 若子View的childDimension是WRAP_CONTENT，则子View的大小不超过父View的大小且mode是AT_MOST

 若父View是UNSPECIFIED，则父View不限制子View大小
    case MeasureSpec.UNSPECIFIED:
    		若子View的childDimension大于0，则表示有确切数值，则子View大小为其本身且mode是EXACTLY
    		若子View的childDimension是MATCH_PARENT，因为父View是UNSPECIFIED，所以子View大小为0
    		若子View的childDimension是WRAP_CONTENT，因为父View是UNSPECIFIED，所以子View大小为0 