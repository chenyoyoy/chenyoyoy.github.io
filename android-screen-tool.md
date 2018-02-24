** 1、问题来了 **
  >在上一篇文章中，我们讲述了一个利用values-wNdp资源路径命名的方式来做分辨率的适配的方案；
     理论上来说，我们只需要配置values-w360dp的资源文件就好了，但是我们无法预估厂商的行为，以及一些怪异的配置，还得考虑兼容240dp ，270dp ，320dp，400dp等设备；
     作为一个高效(偷懒)的程序员，我们明显不会傻乎乎的在每个路径下都计算dimens后加入进去，我们想要一个双击就能生成多套配置文件的方案。

** 2、新的解决方案**
  >解决问题的思路是这个样子的：
   - 根据UI设计图，我们先在values下面写一套默认配置文件dimens.xml；
   - 从dimens.xml 文件夹中读取每个 dimens配置的name 和 value，存到一个list中 ；
   - 根据需要适配的设备配置，重新计算每个dimens 对应的值，形成一个新的值，存到新的newList中；
   - 根据xml文档结构，将newList中的值重新拼接，形成一个字符串，写入到新的目录下的values.xml文件中。

** 3、代码走读**
   >3.1 代码结构
     - Config.java
     - DimensCreator.java
     - DimensParser.java
     - DimensValues.java
     - Test
 >3.2 Test.java--入口主控类，读取默认配置文件，生成多套适配文件；
<code>
DimensParser parser = new DimensParser();
List<DimenValues> list = parser.parse(new FileInputStream(Config.path + File.separator + "dimens.xml"));
DimensCreator creator = new DimensCreator(Config.path, list);creator.createAll();</code>
3.3 Config.java
<code>public class Config {    
//需要适配的设备配置    
public final static int[] supportDevices = {240, 270, 320, 360, 400};    
//dimens.xml 计算基础 360dp    public final static int defaultValue = 360;    
//values 文件夹路径 。默认在此路径下存放dimens.xml    
public final static String path = "D:\\app\\src\\main\\res\\values" ;
}</code>
3.4 DimensParser.java 解析values/dimens.xml文件 ，读取数据到内存中
<code>
private class InnerHandler extends DefaultHandler {   
private DimenValues dimenValues;
    private StringBuilder stringBuilder = new StringBuilder();
    private String tempName;
    @Override
    public void startElement(String uri, String localName, String qName, Attributes attributes) throws SAXException {
        if (qName.equals("dimen")) { 
           tempName = qName; 
           dimenValues = new DimenValues();
            stringBuilder.setLength(0);
            dimenValues.name = attributes.getValue("name");
        } 
   } 
   @Override
    public void endElement(String uri, String localName, String qName) throws SAXException {        if (qName.equals("dimen")) { 
           list.add(dimenValues);
        }
    }
    @Override
    public void characters(char[] ch, int start, int length) throws SAXException {
        stringBuilder.append(ch, start, length); 
       if (tempName != null && tempName.equals("dimen")) {
            dimenValues.value = stringBuilder.toString(); 
       }
    }
}</code>
3.5 DimensCreator.java
3.5.1  xml 文件生成模块：
<code>
private final String xmlHeader = "<?xml version=\"1.0\" encoding=\"utf-8\"?>\n" + "<resources>";
private final String xmlFooter = "</resources>";
private final String DIRTEMPLATE = "values-w%sdp";
private final String dimenTemplate = "<dimen name=\"%s\">%sdp</dimen>";</code>
3.5.2 根据需要适配的设备，遍历生成多个文件
<code>public void createAll() {
    for (int i : Config.supportDevices) {
        //生成values-wNdp的资源目录
        String dir = DIRPath.replace("values", "").trim() + File.separator + String.format(DIRTEMPLATE, "" + i);
        File dirFile = new File(dir);
        if (!dirFile.exists()) {
            dirFile.mkdirs();
        }
        //在values-wNdp的资源目录生成文件
        File file = new File(dirFile.getPath() + File.separator + "dimens.xml");
        //dimens 值计算时候的比例
        float scale = (float) i / Config.defaultValue;
        createSingleFile(file, scale);
    }}</code>
3.5.3 重新计算value ，写入文件中
<code>
//写入xml文件头
String data = xmlHeader;for (DimenValues values : list) {
    String itemValue = values.value;
    //计算新的dp值    
    if (values.value.contains("dp")) {
        float v = Float.parseFloat(values.value.replace("dp", "").trim());
        itemValue = formatDimen(v * scale);
    }
    //拼接每一行的dimen值
    String itemData = String.format(dimenTemplate, values.name, itemValue) + "\n";
    data += itemData;
}
//写入xml文件尾部
data += xmlFooter;FileOutputStream outputStream = null;
ByteArrayInputStream inputStream = null;
try {
    outputStream = new FileOutputStream(file);
    inputStream = new ByteArrayInputStream(data.getBytes());
    byte[] buffer = new byte[inputStream.available()];
    while (inputStream.read(buffer) != -1) {
        outputStream.write(buffer);
    }
    outputStream.flush();
    outputStream.close();
    inputStream.close();
} catch (IOException e) {
    e.printStackTrace();}
}</code>

**4、友情提示**
> 1. 代码中的dimen最好不要用模块相关名称命名，采用中性化命名，比如dimenSize_180dp ;对应w360dP设备的时候等于180dp，w320dP设备的时候等于160dp......
> 2. 完整代码地址：[源码地址](https://github.com/binye33333/android/)
> 3. 如果有特殊需求，请自行阅读代码，然后根据自己需求修改完善
