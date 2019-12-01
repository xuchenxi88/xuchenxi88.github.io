# jQuery DataTables插件 aoColumnDefs和aoColumns

之前工作碰到一个使用DataTables绘制表格的问题，需要将批量数据绘制成表格，并且要按列合并单元格，当有多行数据时出现无法对齐的情况，后来查了一下DataTables插件，找到解决方法

## 发生问题
在使用列合并单元格时，需要使用rowspan参数，但是使用之后，由于没有固定行列数，所以导致多行数据无法对其，生成的表格就没法看了，于是查询了一下DataTalbes的插件参数：

- aoColumnDefs和aoColumns都可以设置列的属性。
- aoColumnDefs设置列的属性时，可以任意指定列，并且不需要给所有列都设置。
- aoColumns设置列时，不可以任意指定列，必须列出所有列。如果某一列不需要设置，则要赋值null。
- 如果aoColumnDefs和aoColumns同时给同一列的同一个属性设置了值，那么aoColumns的优先级要高。

## DataTable初始化参数的详细说明

``` javascript
$(function(){
 var docrTable = $('#aaa').dataTable({  
     "destroy":true,  //不加会报错
     "bProcessing" : true, //DataTables载入数据时，是否显示‘进度’提示  
            "bServerSide" : true, //是否启动服务器端数据导入  
            "bStateSave" : true, //是否打开客户端状态记录功能,此功能在ajax刷新纪录的时候不会将个性化设定回复为初始化状态  
            "bJQueryUI" : true, //是否使用 jQury的UI theme  
            "sScrollY" : 450, //DataTables的高  
            "sScrollX" : 820, //DataTables的宽  
            "aLengthMenu" : [20, 40, 60], //更改显示记录数选项  
            "iDisplayLength" : 40, //默认显示的记录数  
            "bAutoWidth" : false, //是否自适应宽度  
            //"bScrollInfinite" : false, //是否启动初始化滚动条  
            "bScrollCollapse" : true, //是否开启DataTables的高度自适应，当数据条数不够分页数据条数的时候，插件高度是否随数据条数而改变  
            "bPaginate" : true, //是否显示（应用）分页器  
            "bInfo" : true, //是否显示页脚信息，DataTables插件左下角显示记录数  
            "sPaginationType" : "full_numbers", //详细分页组，可以支持直接跳转到某页  
            "bSort" : true, //是否启动各个字段的排序功能  
            "aaSorting" : [[1, "asc"]], //默认的排序方式，第2列，升序排列  
            "bFilter" : true, //是否启动过滤、搜索功能  
                    "aoColumns" : [{  
                        "mDataProp" : "USERID",  
                        "sDefaultContent" : "", //此列默认值为""，以防数据中没有此值，DataTables加载数据的时候报错  
                        "bVisible" : false //此列不显示  
                    }, {  
                        "mDataProp" : "USERNAME",  
                        "sTitle" : "用户名",  
                        "sDefaultContent" : "",  
                        "sClass" : "center"  
                    }, {  
                        "mDataProp" : "EMAIL",  
                        "sTitle" : "电子邮箱",  
                        "sDefaultContent" : "",  
                        "sClass" : "center"  
                    }, {  
                        "mDataProp" : "MOBILE",  
                        "sTitle" : "手机",  
                        "sDefaultContent" : "",  
                        "sClass" : "center"  
                    }, {  
                        "mDataProp" : "PHONE",  
                        "sTitle" : "座机",  
                        "sDefaultContent" : "",  
                        "sClass" : "center"  
                    }, {  
                        "mDataProp" : "NAME",  
                        "sTitle" : "姓名",  
                        "sDefaultContent" : "",  
                        "sClass" : "center"  
                    }, {  
                        "mDataProp" : "ISADMIN",  
                        "sTitle" : "用户权限",  
                        "sDefaultContent" : "",  
                        "sClass" : "center"  
                    }],  
                    "oLanguage": { //国际化配置  
                "sProcessing" : "正在获取数据，请稍后...",    
                "sLengthMenu" : "显示 _MENU_ 条",    
                "sZeroRecords" : "没有您要搜索的内容",    
                "sInfo" : "从 _START_ 到  _END_ 条记录 总记录数为 _TOTAL_ 条",    
                "sInfoEmpty" : "记录数为0",    
                "sInfoFiltered" : "(全部记录数 _MAX_ 条)",    
                "sInfoPostFix" : "",    
                "sSearch" : "搜索",    
                "sUrl" : "",    
                "oPaginate": {    
                    "sFirst" : "第一页",    
                    "sPrevious" : "上一页",    
                    "sNext" : "下一页",    
                    "sLast" : "最后一页"    
                }  
            },  
                      
                    "fnRowCallback" : function(nRow, aData, iDisplayIndex) {  
                        /* 用来改写用户权限的 */  
                        if (aData.ISADMIN == '1')  
                            $('td:eq(5)', nRow).html('管理员');  
                        if (aData.ISADMIN == '2')  
                            $('td:eq(5)', nRow).html('资料下载');  
                        if (aData.ISADMIN == '3')  
                            $('td:eq(5)', nRow).html('一般用户');  
                          
                        return nRow;  
                    },  
                      
                    "sAjaxSource" : "../use/userList.do?now=" + new Date().getTime(),  
                        //服务器端，数据回调处理  
                            "fnServerData" : function(sSource, aDataSet, fnCallback) {  
                                $.ajax({  
                                    "dataType" : 'json',  
                                    "type" : "POST",  
                                    "url" : sSource,  
                                    "data" : aDataSet,  
                                    "success" : fnCallback  
                                });  
                            }  
                });  
                  
                $("#docrevisontable tbody").click(function(event) { //当点击表格内某一条记录的时候，会将此记录的cId和cName写入到隐藏域中  
                    $(docrTable.fnSettings().aoData).each(function() {  
                        $(this.nTr).removeClass('row_selected');  
                    });  
                    $(event.target.parentNode).addClass('row_selected');  
                    var aData = docrTable.fnGetData(event.target.parentNode);  
                      
                    $("#userId").val(aData.USERID);  
                    $("#userN").val(aData.USERNAME);  
                });  
                  
                $('#docrevisontable_filter').html('<span>用户管理列表');  
                $('#docrevisontable_filter').append('   <input type="button" class="addBtn" id="addBut" value="创建"/>');  
                $('#docrevisontable_filter').append('   <input type="button" class="addBtn" id="addBut" value="修改"/>');  
                $('#docrevisontable_filter').append('   <input type="button" class="addBtn" id="addBut" value="删除"/>');  
                $('#docrevisontable_filter').append('</span>');  
        
         });  
 })
```
## 一般通用配置
```javascript
$(function(){
 var docrTable = $('#docrevisontable').dataTable({  
     "destroy":true,  //不加会报错
     "bProcessing" : false, //DataTables载入数据时，是否显示‘进度’提示
     "bInfo" : true, //是否显示页脚信息，DataTables插件左下角显示记录数  
     "sPaginationType" : "full_numbers", //详细分页组，可以支持直接跳转到某页  
     "bSort" : true, //是否启动各个字段的排序功能  
     "aaSorting" : [[0, "asc"]], //默认的排序方式，第n列，升序排列  
     "aLengthMenu" : [10, 20, 50], //更改显示记录数选项  
     "iDisplayLength" : 10, //默认显示的记录数  
     "aoColumns" : [    //有几列就写几个，我有九列，所以是九个
    	 null,
    	 null,
    	 {   //表示第三列
    		"bVisible" :true,  //是否可见
    		"bSearchable": false, //是否参与搜素
    		"bSortable":false,    //是否排序
    		"sWidth":200     //设置宽度
    	 },
    	 null,
    	 null,
    	 null,
    	 null,
    	 null,
    	 null
     ],
 
      "oLanguage": { //显示设置
         "sProcessing" : "正在获取数据，请稍后...",    
         "sLengthMenu" : "显示 _MENU_ 条",    
         "sZeroRecords" : "没有您要搜索的内容",    
         "sInfo" : "从 _START_ 到  _END_ 条记录 总记录数为 _TOTAL_ 条",    
         "sInfoEmpty" : "记录数为0",    
         "sInfoFiltered" : "(全部记录数 _MAX_ 条)",    
         "sInfoPostFix" : "",    
         "sSearch" : "搜索",    
         "oPaginate": {    
             "sFirst" : "第一页",    
             "sPrevious" : "上一页",    
             "sNext" : "下一页",    
             "sLast" : "最后一页"    
         }  
     } 
         });  
 })
 ```