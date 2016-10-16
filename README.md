# -
js原生插件
<!DOCTYPE html>
<html lang="en">

<head>
    <meta charset="UTF-8">
    <title>Drag</title>
    <style>
        body {margin: 0; padding: 0; }
    </style>
</head>

<body id="">
    <div class="drag-area" style="background:#f3f3f3;width:1300px;height:600px;margin:40px;position:relative;">
        <div class="drag-box" style="background:#39f;width:200px;height:300px;margin-left:10px;margin-top:-30px;position:absolute;top:20px;left:100px;"></div>
        <div class="drag-box" style="background:#fc3232;width:300px;height:200px;margin-top:200px;position:absolute;"></div>
    </div>

    <script type="text/javascript">
        var DragAndDrop = function(options) {
            var defaults = {
                el: ".drag-box", //被操作元素
                parent: ".drag-area", //被操作元素定位父元素（dragState为false时不需传该参数）
                dragState: true, //是否允许拖动（需给元素设置position样式）
                scaleState: true, //是否允许缩放
                min: "100,100", //允许缩最小尺寸（scaleState为false时不需传该参数）
                max: "400,400", //允许放最大尺寸（scaleState为false时不需传该参数）
                callback: null
            };

            /*判断是否为空对象*/
            var isEmptyObject = function(target) {
                var key;
                for (key in target) {
                    return !1;
                }
                return !0;
            };
            /*合并对象(浅复制)*/
            var extendObject = function() {
                var arg = arguments,
                    argLen = arg.length,
                    target = arg[0],
                    newObj = arg[1],
                    isOverRide = (arg[argLen - 1] && typeof arg[argLen - 1] === 'boolean') ? arg[argLen - 1] : false,
                    tmp = {},
                    i;
                var isArray = function(obj) {
                        return Object.prototype.toString.call(obj) === '[object Array]';
                    }
                    //排除将要合并的对象是空对象或者是数组
                if (argLen == 0 || (typeof arg[0] !== 'object') || isEmptyObject(newObj) || (isArray(arg[0]))) {
                    return target;
                }
                for (i in target) {
                    if (target.hasOwnProperty(i)) {
                        if (isOverRide) {
                            for (var m in newObj) {
                                if (newObj.hasOwnProperty(m)) {
                                    if (i == m) {
                                        target[i] = newObj[m];
                                    }
                                }
                            }
                        } else {
                            tmp[i] = target[i];
                            for (var k in newObj) {
                                if (newObj.hasOwnProperty(k)) {
                                    if (i == k) {
                                        tmp[i] = newObj[k];
                                    } else {
                                        tmp[k] = newObj[k];
                                    }
                                }
                            }
                        }
                    }
                }
                return isOverRide ? target : tmp;
            };
            var DADoption = extendObject(defaults, arguments[0]); //传入options设置默认参数
            var tag = document.querySelectorAll(DADoption.el);
            var type = 0; //鼠标标识操作为0：拖动/1：缩放
            var dragBefore_X, dragBefore_Y, //拖动之前位置,相对于距其最近定位元素
                dragOffset_X = 0,
                dragOffset_Y = 0, //拖动结束时相对初始位置的偏移量
                draging_X, draging_Y; //拖动过程中的位置，相对于距其最近定位元素，拖动过程实时变化
            var temp_X, temp_Y; //临时差值
            var minTop, minLeft; //最小限制top、left值，拖动不能超出浏览器窗口
            var maxTop, maxLeft; //最大限制top、left值，拖动不能超出浏览器窗口
            var initWidth, initHeith, //缩放时记录元素初始宽高
                currentWidth, currentHeight, //缩放过程中的实时宽高
                zoomWidth = 0,
                zoomHeight = 0; //缩放结束相对于原始变化宽高

            for (var i = 0; i < tag.length; i++) {
                if (DADoption.dragState && !tag[i].style.position) { //必须给待操作元素设置position样式
                    return;
                }
                //拖动鼠标样式
                if (DADoption.dragState) {
                    tag[i].style.cursor = "move";
                }
                if (DADoption.scaleState) {

                    //缩放角标
                    var scaleNode = document.createElement("span");
                    scaleNode.className = "scaleIcon";
                    scaleNode.style.cssText = "display:inline-block;width: 5px;height: 5px; background-color: #333;border: 1px #fff solid;position: absolute;right: 0px;bottom: 0px;";
                    tag[i].appendChild(scaleNode);
                    //缩放监听
                    for (var r = 0; r < document.querySelectorAll(".scaleIcon").length; r++) {
                        var item = document.querySelectorAll(".scaleIcon")[i];
                        item.onmouseover = function(e) {
                            this.style.cursor = "se-resize";
                            this.onmousedown = function(e) {
                                type = 1;
                            }
                        }
                        item.onmouseout = function(e) {
                            type = 0;
                        }
                    }
                }


                tag[i].onmousedown = function(e) {
                    document.onselectstart = function() {
                        return false;
                    };

                    var tempEle = this;
                    //兼容浏览器获取event值不一致
                    var event = e || window.Event;
                    if (DADoption.dragState && type == 0) { //拖动
                        var elParent = e.target.parentNode; //获取当前选中元素的定位父元素
                        if (DADoption.parent.indexOf(".") >= 0) {
                            while (elParent.tagName != "BODY" && elParent.className.indexOf(DADoption.parent.replace(/^\.{1}/, "")) < 0) {
                                elParent = elParent.parentNode;
                            }
                        } else if (DADoption.parent.indexOf("#") >= 0) {
                            while (elParent.tagName != "BODY" && elParent.getAttribute("id") != DADoption.parent.replace(/^\#{1}/, "")) {
                                elParent = elParent.parentNode;
                            }
                        }
                        draging_X = dragBefore_X = this.offsetLeft;
                        draging_Y = dragBefore_Y = this.offsetTop;
                        temp_X = event.pageX - dragBefore_X;
                        temp_Y = event.pageY - dragBefore_Y;
                        minTop = -elParent.getBoundingClientRect().top;
                        minLeft = -elParent.getBoundingClientRect().left;
                        maxTop = document.body.scrollHeight - (this.offsetHeight) - elParent.getBoundingClientRect().top;
                        maxLeft = document.body.scrollWidth - (this.offsetWidth) - elParent.getBoundingClientRect().left;
                        document.onmousemove = function(e) {
                            e = e || window.Event;
                            draging_X = e.pageX - temp_X;
                            draging_Y = e.pageY - temp_Y;
                            if (draging_X <= minLeft) {
                                draging_X = minLeft;
                            }
                            if (draging_Y <= minTop) {
                                draging_Y = minTop;
                            }
                            if (draging_X >= maxLeft) {
                                draging_X = maxLeft;
                            }
                            if (draging_Y >= maxTop) {
                                draging_Y = maxTop;
                            }

                            dragOffset_X = draging_X - dragBefore_X;
                            dragOffset_Y = draging_Y - dragBefore_Y;
                            if (tempEle.style.marginLeft) {
                                draging_X = draging_X - parseInt(document.defaultView.getComputedStyle(tempEle, null).marginLeft);
                            }
                            if (tempEle.style.marginTop) {
                                draging_Y = draging_Y - parseInt(document.defaultView.getComputedStyle(tempEle, null).marginTop);
                            }
                            tempEle.style.left = draging_X + "px";
                            tempEle.style.top = draging_Y + "px";
                        }
                    } else if (DADoption.scaleState && type == 1) { //缩放
                    currentWidth = initWidth = document.defaultView.getComputedStyle(this, null).width;
                        currentHeight = initHeith = document.defaultView.getComputedStyle(this, null).height;

                        document.onmousemove = function(e) {
                            e = e || window.Event;
                            currentWidth = parseInt(initWidth) + (e.pageX - event.pageX);
                            currentHeight = parseInt(initHeith) + (e.pageY - event.pageY);
                            if (currentWidth < DADoption.min.split(",")[0]) {
                                currentWidth = DADoption.min.split(",")[0];
                            }
                            if (currentHeight < DADoption.min.split(",")[1]) {
                                currentHeight = DADoption.min.split(",")[1];
                            }
                            if (currentWidth > DADoption.max.split(",")[0]) {
                                currentWidth = DADoption.max.split(",")[0];
                            }
                            if (currentHeight > DADoption.max.split(",")[1]) {
                                currentHeight = DADoption.max.split(",")[1];
                            }
                            tempEle.style.width = currentWidth + "px";
                            tempEle.style.height = currentHeight + "px";
                            zoomWidth = currentWidth - parseInt(initWidth);
                            zoomHeight = currentHeight - parseInt(initHeith);
                        }
                    }
                }

            }
            document.onmouseup = function() {

                if (DADoption.callback) { //回调函数返回拖动相对偏移量和缩放后改变的大小
                    DADoption.callback.call(this, {
                        offset_X: dragOffset_X,
                        offset_Y: dragOffset_Y,
                        zoom_Width: zoomWidth,
                        zoom_Height: zoomHeight
                    });
                }
                document.onmousemove = null;
                dragOffset_X = 0;
                dragOffset_Y = 0;
                zoomWidth = 0;
                zoomHeight = 0;
            }
        };
        DragAndDrop({
            dragState: true, //是否允许拖动（需给元素设置position样式）
            scaleState: true, //是否允许缩放
            min: "100,200", //允许缩最小尺寸（scaleState为false时不需传该参数）
            max: "400,400", //允许放最大尺寸（scaleState为false时不需传该参数）
            callback: function(res) {
                console.log(res);
            }
        });
    </script>
</body>
