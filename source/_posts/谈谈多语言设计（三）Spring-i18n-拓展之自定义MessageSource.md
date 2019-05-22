title: 谈谈多语言设计（三）Spring-i18n 拓展之自定义MessageSource
author: Jiacan Liao
tags:
  - i18n
  - 多语言
categories:
  - 架构总结
date: 2018-04-13 16:08:00
---
&emsp;&emsp;Spring框架中有两个MessageSource的实现，分别是ResourceBundleMessageSource和ReloadableResourceBundleMessageSource，前者第一次初始化就固定下来，后者可以根据配置文件是否发生变进行更新。

&emsp;&emsp;对于Web网页（UI上的文案），这种配置在配置文件的方式是可以接受的，因为一般这些文案都是固定的。但是对于一些需要动态新增配置的场景显然就是不适合了，比如商品信息，抽奖的奖品，直播间的道具，礼物等，这些都是根据运营人员需要动态调整的，显然需要把配置存储在数据库中。

&emsp;&emsp;在写这部分的实现的时候，参考了一个开源项目，[https://github.com/synyx/messagesource](https://github.com/synyx/messagesource)。
感兴趣的同学，可以在我的Github查看完整的代码。[https://github.com/liaojiacan/spring-i18n-support](https://github.com/liaojiacan/spring-i18n-support)


## UML

![upload successful](/images/pasted-3.png)

##  关于MessageSourceProvider
&emsp;&emsp; 在MessageSource的实现中 ，抽出一个Provider层将存储介质解耦，可以在最后的应用中，选择使用JDBC还是Redis还是远程的配置服务中心的实现。
```
public interface MessageSourceProvider {

	List<MessageEntry> load();

	int addMessage(Locale locale,String code,String type,String message);

	int updateMessage(Locale locale,String code,String type,String message);

	int deleteMessage(Locale locale,String code);

}

```
## JdbcMessageSoucreProvider
&emsp;&emsp; 因为只是简单的对数据进行CURD，所以采用JdbcTemple的方式减少相关的依赖，采用java原生的jdbc也是可以的。

```
-- ----------------------------
-- Table structure for i18n_message
-- ----------------------------
DROP TABLE IF EXISTS `i18n_message`;
CREATE TABLE `i18n_message` (
  `code` varchar(250) NOT NULL COMMENT 'mapping code',
  `locale` varchar(100) NOT NULL COMMENT 'language tag',
  `type` varchar(100) DEFAULT NULL COMMENT 'type for group',
  `message` text NOT NULL COMMENT 'message content',
  `create_time` datetime DEFAULT CURRENT_TIMESTAMP COMMENT 'create time',
  `update_time` timestamp NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT 'last modify time',
  PRIMARY KEY (`code`,`locale`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8 COMMENT='i18n message data';

SET FOREIGN_KEY_CHECKS = 1;
```

```
public class JdbcMessageSoucreProvider implements MessageSourceProvider {

	private JdbcTemplate jdbcTemplate;

	protected static final String QUERY_TPL_INSERT_MESSAGE_ENTRY =
			"INSERT INTO %s (%s, %s, %s, %s) VALUES (?, ?, ?, ?)";
	protected static final String QUERY_TPL_DELETE_MESSAGE_ENTRY = "DELETE FROM %s WHERE %s = ? and %s= ?";
	protected static final String QUERY_TPL_SELECT_MESSAGE_ENTRIES = "SELECT %s,%s,%s,%s FROM %s";
	protected static final String QUERY_TPL_UPDATE_MESSAGE_ENTRY = "UPDATE %s set %s=?,%s=?,%s=? WHERE %s=? and %s=?";

	private String localeColumn = "locale";
	private String typeColumn = "type";
	private String codeColumn = "code";
	private String messageColumn = "message";
	private String tableName = "i18n_message";
	private String delimiter = "`";


	@Override
	public List<MessageEntry> load() {
		// @formatter:off
		String sql = String.format(getQueryTplSelectMessageEntries(),
				addDelimiter(getCodeColumn()),
				addDelimiter(getLocaleColumn()),
				addDelimiter(getTypeColumn()),
				addDelimiter(getMessageColumn()),
				addDelimiter(getTableName()));
		return jdbcTemplate.query(sql,new BeanPropertyRowMapper(MessageEntry.class));
		// @formatter:on
	}

	@Override
	public int addMessage(Locale locale, String code, String type, String message) {
		// @formatter:off
		String sql = String.format(getQueryTplInsertMessageEntry(),
				addDelimiter(getTableName()),
				addDelimiter(getLocaleColumn()),
				addDelimiter(getCodeColumn()),
				addDelimiter(getTypeColumn()),
				addDelimiter(getMessageColumn()));
		// @formatter:on
		return jdbcTemplate.update(sql,locale.toLanguageTag(),code,type,message);
	}


	@Override
	public int updateMessage(Locale locale, String code, String type, String message) {
		// @formatter:off
		String sql = String.format(getQueryTplUpdateMessageEntry(),
				addDelimiter(getTableName()),
				addDelimiter(getLocaleColumn()),
				addDelimiter(getTypeColumn()),
				addDelimiter(getMessageColumn()),
				addDelimiter(getCodeColumn()),addDelimiter(getLocaleColumn()));
		// @formatter:on
		return jdbcTemplate.update(sql,locale.toLanguageTag(),type,message,code,locale.toLanguageTag());
	}

	@Override
	public int deleteMessage(Locale locale, String code) {
		String sql = String.format(getQueryTplDeleteMessageEntry(),addDelimiter(getTableName()),addDelimiter(getCodeColumn()),addDelimiter(getLocaleColumn()));
		return jdbcTemplate.update(sql,code,locale.toLanguageTag());
	}

	/**
	 * Method that "wraps" a field-name (or table-name) into the delimiter.
	 * @param name the name of the field/table
	 * @return the wrapped field/table
	 */
	protected String addDelimiter(String name) {
		return String.format("%s%s%s", delimiter, name, delimiter);
	}


	public JdbcTemplate getJdbcTemplate() {
		return jdbcTemplate;
	}

	public void setJdbcTemplate(JdbcTemplate jdbcTemplate) {
		this.jdbcTemplate = jdbcTemplate;
	}

	public static String getQueryTplInsertMessageEntry() {
		return QUERY_TPL_INSERT_MESSAGE_ENTRY;
	}

	public static String getQueryTplDeleteMessageEntry() {
		return QUERY_TPL_DELETE_MESSAGE_ENTRY;
	}

	public static String getQueryTplSelectMessageEntries() {
		return QUERY_TPL_SELECT_MESSAGE_ENTRIES;
	}

	public static String getQueryTplUpdateMessageEntry() {
		return QUERY_TPL_UPDATE_MESSAGE_ENTRY;
	}

	public String getLocaleColumn() {
		return localeColumn;
	}

	public void setLocaleColumn(String localeColumn) {
		this.localeColumn = localeColumn;
	}

	public String getTypeColumn() {
		return typeColumn;
	}

	public void setTypeColumn(String typeColumn) {
		this.typeColumn = typeColumn;
	}

	public String getCodeColumn() {
		return codeColumn;
	}

	public void setCodeColumn(String codeColumn) {
		this.codeColumn = codeColumn;
	}

	public String getMessageColumn() {
		return messageColumn;
	}

	public void setMessageColumn(String messageColumn) {
		this.messageColumn = messageColumn;
	}

	public String getTableName() {
		return tableName;
	}

	public void setTableName(String tableName) {
		this.tableName = tableName;
	}

	public String getDelimiter() {
		return delimiter;
	}

	public void setDelimiter(String delimiter) {
		this.delimiter = delimiter;
	}
```

## RefreshableMessageSource
&emsp;&emsp;RefreshableMessageSource的实现相对简单，在初始化的时候将MessageSourceProvider的数据解析成MessageFormat存在Map中，解析的时候根据code和locale索引到对应的MessageFormat。

```
/**
 * @author liaojiacan https://github.com/liaojiacan
 */
public class RefreshableMessageSource extends AbstractMessageSource implements Refreshable,InitializingBean{

	private MessageSourceProvider provider;

	/**
	 * Setting : return origin code when the message not found.
	 */
	protected Boolean returnUnresolvedCode = false;

	/**
	 * The MessageFormat cache
	 */
	private Map<String,Map<Locale,MessageFormat>> messageEntryMap = Collections.emptyMap();

	public RefreshableMessageSource(MessageSourceProvider provider) {
		this.provider = provider;
	}

	@Override
	public void refresh(){
		List<MessageEntry> messageEntries = provider.load();
		if(!CollectionUtils.isEmpty(messageEntries)){
			final Map<String,Map<Locale,MessageFormat>> finalMap = new HashMap<>();
			messageEntries.forEach(messageEntry -> {
				String code  = messageEntry.getCode();
				Locale locale = Locale.forLanguageTag(messageEntry.getLocale());
				Map<Locale, MessageFormat> localeMapping = finalMap.get(code);
				if(localeMapping == null){
					localeMapping = new HashMap<>();
					finalMap.put(code,localeMapping);
				}
				localeMapping.put(locale,createMessageFormat(messageEntry.getMessage(),locale));

			});
			messageEntryMap = finalMap;
		}
	}

	@Override
	protected MessageFormat resolveCode(String code, Locale locale) {
		Map<Locale, MessageFormat> localeMessageMap = messageEntryMap.get(code);
		if(localeMessageMap != null ){
			MessageFormat mf = localeMessageMap.get(locale);
			if(mf!=null){
				return  mf;
			}
		}
		if(returnUnresolvedCode){
			return createMessageFormat(code,locale);
		}
		return null;
	}

	@Override
	public void afterPropertiesSet() throws Exception {
		this.refresh();
	}

	public MessageSourceProvider getProvider() {
		return provider;
	}

	public void setProvider(MessageSourceProvider provider) {
		this.provider = provider;
	}

	public Boolean getReturnUnresolvedCode() {
		return returnUnresolvedCode;
	}

	public void setReturnUnresolvedCode(Boolean returnUnresolvedCode) {
		this.returnUnresolvedCode = returnUnresolvedCode;
	}
}
```