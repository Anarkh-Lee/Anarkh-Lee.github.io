---
layout: post
title: '通过filter实现AngularJs的girder的cellFilter自定义代码项'
date: 2021-01-04
author: Anarkh-Lee
cover: 'http://on2171g4d.bkt.clouddn.com/jekyll-banner.png'
tags: AngularJs








---

> 开发过程中，为了不在数据库中增加代码项，有时需要自定义cellFilter中的代码项仅为显示作用。

开发过程中，框架中对于代码项的选择已经封好了公共方法，在gird中对字段的cellFilter属性进行`cellFilter: "selectCodeName:\'SP_WJRC04\'"`操作即可显示数据库中对应`SP_WJRC04`的代码值。但是，有的时候需要对代码值进行合并（比如数据库中字段`SHZT`代码项`01申请`，`02审核通过`，`03审核不通过`，`04审批通过`，`05审批不通过`；现将其合并：01,02作为待审批，04审批通过，05审批不通过），又不想新增数据库代码项，编写如下伪代码提供参考：

```javascript
/**
 * @author  Anarkh
 * @date  2020/01/04 9:31
 * @description
 */
'use strict';
angular.module('com.anarkh.businessModule')

    .controller('businessCtl', ['$scope', 'businessService', 'messageService','$filter',
        function ($scope, businessService, messageService,$filter) {

            $scope.initGrid1 = function () {
                $scope.columns1 = [{
                    field : 'ywzt',
                    displayName : '业务状态',
                    cellFilter: 'ywztFilter',
                    width : '130'
                },{
                    field: 'sfz',
                    displayName: '身份证号',
                    width: '180',
                    enableCellEdit: false
                }, {
                    field: 'wjrc04',
                    displayName: '学历',
                    cellFilter: "selectCodeName:\'SP_WJRC04\'",
                    width: '130'
                }];

                $scope.applyListOptions = {
                    columnDefs: $scope.columns1,
                    data: 'gridListinfo',
                    showGridFooter : true,
                    enableGridMenu : true,
                    enableColumnMenus: false,
                    enableSorting: true,
                    enableRowSelection: true,
                    multiSelect: true,
                    paginationPageSizes: [10],
                    paginationPageSize: 10,
                    enableFullRowSelection: true, // 是否点击行任意位置后选中,default为false,当为true时,checkbox可以显示但是不可选中
                    exporterMenuPdf: false,
                    exporterOlderExcelCompatibility : true,
                    onRegisterApi: function (gridApi) {
                        $scope.gridApi1 = gridApi;
                        gridApi.selection.on.rowSelectionChanged($scope, function (row) {
                            $scope.selectedRow = row.isSelected ? row.entity : null;
                        });
                    },
                    rowHeight: 28
                };
            };



            $scope.initPage1 = function () {
                $scope.initGrid1();
                $scope.gridListinfo = [];
                $scope.doQuery();
            };

            $scope.doQuery = function(){
                businessService.doQuery($scope.query).$promise.then(function(data) {
                    $scope.gridListinfo = data;
                }, function(err) {
                    $scope.messageBox.showError(err.data.detail);
                });
            }

        }
    ])
    .filter('ywztFilter', function () {

        var list = [{value: '01', name: '待审批'},
            {value: '02', name: '待审批'},
            {value: '04', name: '审批通过'},
            {value: '05', name: '审批不通过'}];
        return function (value) {

            //错误写法：
            //由于forEach不支持return跳出循环所以这么写有问题。
            // angular.forEach(list, function (data) {
            //     if (data.value === value) {
            //         return data.name;
            //     }
            // });

            for (var item in list) {
                if (list[item].value === value) {
                    return list[item].name;
                }
            }

        };

    });
```

效果：

![](.\img\AngularJs\cellFilter01.png)





<iframe type="text/html" width="100%" height="385" src="http://www.youtube.com/embed/gfmjMWjn-Xg" frameborder="0"></iframe>