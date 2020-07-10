---
layout: post
title: 'AngularJs中ui-bootstrap模态框总结'
date: 2020-07-02
author: Anarkh-Lee
cover: 'http://on2171g4d.bkt.clouddn.com/jekyll-banner.png'
tags: AngularJs


---

> 开发过程中使用AngularJs模态框的一些总结

只是为了说明ui-bootstrap模态框的用法，因此只显示主要的代码。

**1.原生ui-bootstrap模态框的使用**

主页面Html

```html
<button class="btn btn-success" style="width: 100px" ng-click="modalTest()">测试模态框</button>
```

主页面js

```javascript
.controller('MainCtrl', ['$scope', '$uibModal',
    function ($scope, $uibModal) {

        $scope.modalTest=function(){
            var modalInstance=$uibModal.open({
                templateUrl: 'modules/xxx/xxx/xxx/view/test.html',  //载入的html文件
                controller: 'ModalTestCtrl',  //为载入的文件定义一个控制器
                size: 'lg',  // size ： lg sm 两值
                keyboard: false,//boolean默认值：true。当按下 escape 键时关闭模态框，设置为false 时则按键无效。
                backdrop : true,//boolean 或 string 'static'默认值：true。指定一个静态的背景或为false时，当用户点击模态框外部时不会关闭模态框。
                resolve: {   //resolve是成功创建模态框时，将有效数据传给模态框的控制器，模态框的控制通过注入的形式获取，这里传送了一个item的值；
                    item: function () {
                        // return $scope.item;
                        return '传值成功！';
                    }
                }
            });

            modalInstance.opened.then(function(){//模态窗口打开之后执行的函数，用的比较少，可以注掉
                console.log('modal is opened');
                alert('模态窗口打开之后执行的函数');
            });

            modalInstance.result.then(function (data) {  //配合模态框模块执行完毕，成功关闭后执行的回调函数。selected是模态框传过来的值。
                alert(data);//成功关闭后执行。$modalInstance.close('anarkh-lee');
            }, function () {
                alert('error');//取消关闭后执行。$modalInstance.dismiss('cancel');　
            });
        }

}])
```

模态框Html

```html
<div class="modal-header">
    <table style="width: 100%">
        <tr>
            <td align="left">
                <h4>模态框测试</h4>
            </td>
        </tr>
    </table>
</div>
<div class="panel panel-default">
    <div class="panel-body">
        <form name="queryForm" class="form-horizontal formCustomSetting normalForm">
            <h1>模态框</h1>
            <div class="form-group">
                <button class="btn btn-primary" type="button" ng-click="ok()">OK</button>
                <button class="btn btn-warning" type="button" ng-click="cancel()">Cancel</button>
            </div>
        </form>
    </div>
</div>
```

模态框js

```javascript
.controller('ModalTestCtrl', ['$scope', '$modalInstance','item',
    function ($scope, $modalInstance,item) {
        var doInit = function() {
            alert(item);//接收参数
        };
        doInit();
        $scope.ok = function () {
            $modalInstance.close('anarkh-lee');  //close方法会将参数值回调返回。
        };
        $scope.cancel = function () {
            $modalInstance.dismiss('cancel');  //关闭模态框，也会执行回调。
        };
    }])
```



**2.开发过程中用到的一种封装了一层的模态框的使用**

这里只展示主页面js和模态框js

主页面js

```javascript
$scope.openInstruction = function() {
    var modalDefault = {
        backdrop : true,
        keyboard : true,
        modalFade : true,
        size : 'lg',
        scope : $scope,
        controller : 'modalCtl',
        templateUrl : 'modules/xxxx/xxx/xxx/view/rcInstruction.html'
    };
    var modalOptions = {
        ywlb:'1',
        callBack : function() {//这里可以接参数
            //执行回调函数
        }
     };
     messageService.showModal(modalDefault, modalOptions);
};
```

模态框js

```javascript
.controller('modalCtl', ['$scope', '$modalInstance', 'modalOptions','rcCommonService',
    function ($scope, $modalInstance, modalOptions,rcCommonService) {
		var doInit = function() {
        $scope.tfmsg = {};
        rcCommonService.queryIns({ywlb:modalOptions.ywlb}).$promise.then(function(data) {
            if(data.txsm != null){
                $scope.tfmsg.txsm = data.txsm;
            }
        	}, function(err) {
            	$scope.messageBox.showError(err.data.detail);
        	});
    	};
    	doInit();
		$scope.doClose = function() {
            $modalInstance.dismiss('cancel');
        };
    }])
```



<iframe type="text/html" width="100%" height="385" src="http://www.youtube.com/embed/gfmjMWjn-Xg" frameborder="0"></iframe>