---
layout: post
title: 'Word导出:poi-tl'
date: 2021-12-27
author: Anarkh-Lee
cover: 'http://on2171g4d.bkt.clouddn.com/jekyll-banner.png'
tags: 轮子














---

> Word导出工具：poi-tl

### 1.poi和poi-tl的官网api

poi：

http://deepoove.com/poi-tl/apache-poi-guide.html

poi-tl:

http://deepoove.com/poi-tl/

### 2.maven依赖

```xml
<dependency>
	<groupId>org.apache.poi</groupId>
	<artifactId>poi</artifactId>
	<version>4.1.1</version>
</dependency>

<dependency>
	<groupId>org.apache.poi</groupId>
	<artifactId>poi-ooxml</artifactId>
	<version>4.1.1</version>
</dependency>

<dependency>
	<groupId>org.apache.poi</groupId>
	<artifactId>poi-ooxml-schemas</artifactId>
	<version>4.1.1</version>
</dependency>
<dependency>
    <groupId>com.deepoove</groupId>
    <artifactId>poi-tl</artifactId>
    <version>1.7.3</version>
</dependency>
```

jdk使用的是1.8

poi-tl不同版本需要不同的poi版本与jdk版本。可以参照官网：

![](.\img\轮子\poi-tl.png)

如果版本不匹配会报错（使用1.8.2报错日志待截取），比如



### 3.poi-tl使用实例

前台：

```javascript
//直接window.open即可
function courtRecord(){
    window.open(neu.baseUrl+"/issueDetail/downloadCourtRecord?trialid=" + trialId);
}
```

**后台：**

模板（bweam.docx）：

```word
***劳动人事争议仲裁委员会
庭 审 笔 录

时  间：{{date}}
地  点：{{address}}
仲裁员：{{zcy}}
书记员：{{sjy}}
案  号：{{ah}}
案  由：{{ay}} 
申请人：{{sqr}}
被申请人：{{bsqr}} 
委托代理人：{{wtdlr}} 
{{quesAnswList}}

```

controller:

```java
@ApiOperation("庭审笔录")
@GetMapping(value = "/downloadCourtRecord")
public void downloadCourtRecord(@RequestParam(value = "trialid") String trialid,HttpServletResponse response) {
    issueDetailService.downloadCourtRecord(trialid,response);
}
```

Impl:

```java
@Override
public void downloadCourtRecord(String trialId,HttpServletResponse response) {
    Map<String, Object> data = new HashMap<>();

    LambdaQueryWrapper<SynBaseinfo> wrapper1 = new LambdaQueryWrapper<>();
    wrapper1.eq(SynBaseinfo::getTrialid, trialId);
    List<SynBaseinfo> baseinfoList = this.synBaseinfoDAO.selectList(wrapper1);
    if(baseinfoList.size()>0){
        data.put("date",new SimpleDateFormat("yyyy-MM-dd HH:mm").format(baseinfoList.get(0).getStart_time()));//时间
        data.put("address","");//地点
        data.put("zcy",baseinfoList.get(0).getZcyname());//仲裁员
        data.put("sjy",baseinfoList.get(0).getSjyname());//书记员
        data.put("ah",baseinfoList.get(0).getCasenum());//案号
        data.put("ay",baseinfoList.get(0).getAy());//案由
        data.put("sqr",baseinfoList.get(0).getSqr());//申请人
        data.put("bsqr",baseinfoList.get(0).getBsqr());//被申请人
        data.put("wtdlr","");//委托代理人
    }


    List<SynIssueDTO> quansList = this.queryIssueListByTrialId(trialId);
    //循环数据使用ListRenderPolicy插件
    ListRenderPolicy policy = new ListRenderPolicy();
    //官方api中给的是：Configure config = Configure.builder().bind("title", new DocumentRenderPolicy()).build();
    //这里Configure.builder()不好用，遂使用Configure.newBuilder()
    Configure config = Configure.newBuilder().bind("quesAnswList", policy).build();
    List<WordDTO> detailList = new ArrayList<>();
    if(quansList.size()>0){
        for (int i = 0; i < quansList.size(); i++) {
            WordDTO quansDto = new WordDTO();
            List<String> answerList = new ArrayList<>();
            quansDto.setQuestion(i + 1 + "." + quansList.get(i).getIssuename());
            if(quansList.get(i).getAnswerDTOList().size()>0){
                for (int j = 0; j < quansList.get(i).getAnswerDTOList().size(); j++) {
                    String type = quansList.get(i).getAnswerDTOList().get(j).getAnswertype();
                    //202111223 先取庭审笔录姓名，如果没有再取姓名
                    String name = quansList.get(i).getCaseDTOList().get(j).getNote_name();
                    if(StringUtils.isBlank(name)){
                        name = quansList.get(i).getAnswerDTOList().get(j).getOperatname();
                    }
                    String answer = quansList.get(i).getAnswerDTOList().get(j).getAnswer();
                    switch (type){
                        case "01":
                            type = "申请人";
                            break;
                        case "02":
                            type = "申请代理人";
                            break;
                        case "03":
                            type = "被申请人";
                            break;
                        case "04":
                            type = "被申请代理人";
                            break;
                        default:
                            type = "";
                    }

                    String answerItem = type + " " + name + "：" + answer;
                    answerList.add(answerItem);
                }
            }
            quansDto.setAnswerlist(answerList);
            detailList.add(quansDto);
        }
    }

    List<Object> list = new ArrayList<>();
    Style answerStyle = new Style();
    answerStyle.setFontSize(12);
    detailList.forEach(wordVO -> {
        list.add(new TextRenderData(wordVO.getQuestion()));
        //此处循环放数据
        if (CollectionUtils.isNotEmpty(wordVO.getAnswerlist())) {
            for (int i = 0; i < wordVO.getAnswerlist().size(); i++) {
                list.add(new TextRenderData(wordVO.getAnswerlist().get(i),answerStyle));
            }
        }
    });
    data.put("quesAnswList", list);

    //设置response
    response.setContentType("application/msword");
    response.setHeader("Content-disposition", "attachment;filename=CourtRecord.docx");
    //取绝对路径
    String path = ClassUtils.getDefaultClassLoader().getResource("").getPath();
    XWPFTemplate template = XWPFTemplate.compile(path+"/file/bweam.docx",config)
        .render(data);
    //写在()里try with resources自动关流
    try (OutputStream os = response.getOutputStream();
         BufferedOutputStream bos = new BufferedOutputStream(os);){
        template.write(bos);
    } catch (Exception e) {
        e.printStackTrace();
    }finally {
        //使用poitl工具类关流
        PoitlIOUtils.closeQuietlyMulti(template);
    }
}
```





**导出结果：**

![](.\img\轮子\word.png)

### 4.poi-tl遍历图片

**使用版本：**

```xml
<dependency>
    <groupId>com.deepoove</groupId>
    <artifactId>poi-tl</artifactId>
    <version>1.10.1</version>
</dependency>
```

**代码：**

```java
@Data
public class WordVO {
    private List<PictureRenderData> picture;

    private String problem;

    private String reason;
}
```

```java
import com.deepoove.poi.XWPFTemplate;
import com.deepoove.poi.config.Configure;
import com.deepoove.poi.data.PictureRenderData;
import com.deepoove.poi.data.TextRenderData;
import com.deepoove.poi.policy.ListRenderPolicy;
import com.twy.entity.WordVO;
import org.apache.commons.collections4.CollectionUtils;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;

import java.io.IOException;
import java.io.InputStream;
import java.util.ArrayList;
import java.util.HashMap;
import java.util.List;
import java.util.Map;

@RestController
public class TestController {

    @GetMapping("/download")
    public void download() throws IOException {
        Map<String, Object> map = new HashMap<String, Object>();
        InputStream inputStream = Thread.currentThread().getContextClassLoader()
                .getResourceAsStream("template/test.docx");
        ListRenderPolicy policy = new ListRenderPolicy();
        Configure config = Configure.builder().bind("pictures", policy).build();
        List<WordVO> detailList = new ArrayList<>();
        WordVO one = new WordVO();
        WordVO two = new WordVO();
        one.setProblem("问题测试1");
        two.setProblem("问题测试2");
        one.setReason("原因测试1");
        two.setReason("原因测试2");
        List<PictureRenderData> pList1 = new ArrayList<>();
        List<PictureRenderData> pList2 = new ArrayList<>();
        String location = System.getProperty("user.dir") + "/poi-tl/src/main/resources/";
        pList1.add(new PictureRenderData(100, 120, location + "img/1.jpg"));
        pList1.add(new PictureRenderData(100, 120, location + "img/2.jpg"));
        pList2.add(new PictureRenderData(100, 120, location + "img/3.jpg"));
        one.setPicture(pList1);
        two.setPicture(pList2);
        detailList.add(one);
        detailList.add(two);
        List<Object> list = new ArrayList<>();
        detailList.forEach(wordVO -> {
            list.add(new TextRenderData(wordVO.getProblem()));
            list.add(new TextRenderData(wordVO.getReason()));
            if (CollectionUtils.isNotEmpty(wordVO.getPicture())) {
                list.addAll(wordVO.getPicture());
            }
        });
        map.put("pictures", list);
        XWPFTemplate template = XWPFTemplate.compile(inputStream, config).render(map);
        template.writeToFile(location + "result/test.docx");
    }
}
```

**目录结构：**

![](.\img\轮子\mljg.png)

**word模板：**

![](.\img\轮子\mb.png)

**导出结果：**

![](.\img\轮子\result.png)







<iframe type="text/html" width="100%" height="385" src="http://www.youtube.com/embed/gfmjMWjn-Xg" frameborder="0"></iframe>