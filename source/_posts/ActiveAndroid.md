---
title: ActiveAndroid
date: 2016-05-07 23:37:38
tags: 第三方开源库
---

### 总流程：
	ActiveAndroid.initialize(this,true); 入口
  
- initialize有三个构造方法 
	
	 initialize(Context context)
	 initialize(Configuration configuration)
	initialize(Context context, boolean loggingEnabled)

- 这三个构造方法内部其实都是调用了一个方法

	public static void initialize(Configuration configuration, boolean loggingEnabled) {
		// Set logging enabled first
		//是否打印log信息  
		setLoggingEnabled(loggingEnabled);
		//初始化Cache
		Cache.initialize(configuration);
	}


- cache中的初始化分为如下几步: 

	1. 创建ModelInfo，modelInfo又会创建关于tableInfo和TypeSerializer的信息 
	2. 创建DatabaseHelper，这个DatabaseHelper就是继承自SQLiteOpenHelper，
	3. 创建一个LruCache用来对数据库对象进行缓存，加快存取操作 
	4. 调用openDatabase(),这个函数实际上调用了sDatabaseHelper.getWritableDatabase()， 
	5. ModelInfo储存了每个需要储存的类与TableInfo的映射关系。每当用户要对要储存的类进行读/写操作时，就需要从mTableInfos这个map中找到属于自己的TableInfo，然后根据TableInfo中信息进行相关操作。
	6. TableInfo 就是通过穿过来的Class文件 获取里面 @Table@Column 的值 然后存储到Map集合中 。
	7. DatabaseHelper 其实就是继承了 SQLiteOpenHelper
	8. 
	在onCreate的时候 调用了一个方法  这个方法 先获取 存有所有的Tableinfo的这个Map集合 然后遍历 这个集合 然后又调用一个创建表的方法 把TableInfo传入进去
	在里面在遍历TableInfo这一个表的所有字段，然后执行sql语句，把这些字段动态的添加到sql语句中，从而达到创建表的操作。

-总的来说：Application初始化 调用一个Cache的初始化，Cache里面又初始化了几个东西
		一个是ModelInfo，存储所有的 TableInfo的Map集合
		一个是DatabaseHelper，继承了SQLiteOpenHelper的一个类，在里面遍历获取到集合中的内容，然后和固定的sql语句组合成完整的语句，最终完成创建数据库表的操作.
	
---
Cache.initialize这个方法：

	  //获取context；
	sContext = configuration.getContext();
	  //创建一个ModelInfo 获取自定义的model信息 
	   获取设置的table名和columns名 存入到一个map集合中
	sModelInfo = new ModelInfo(configuration);
	  //创建一个DataBaseHeleper  获取map集合  把里面的自定义的名字啊 列啊  都获取出来 然后底部还是通过一个继承SQLiteOpenHelper的类 把这些值都填入   进去 最后创建数据库
	sDatabaseHelper = new DatabaseHelper(configuration);


sModelInfo = new ModelInfo(configuration)
	
	
	//内部创建了一个Map集合
	private Map<Class<? extends Model>, TableInfo> mTableInfos = new HashMap<Class<? extends Model>, TableInfo>();
	//创建一个list集合 获取configuration那个装有继承model类的list集合
	final List<Class<? extends Model>> models = configuration.getModelClasses();
		if (models != null) {
			for (Class<? extends Model> model : models) {
				如果model不等于空 就遍历获取出来的list集合
				然后存入到之前创建的那个 Map集合
		
				mTableInfos.put(model, new TableInfo(model));

			}
		}

mTableInfos.put(model, new TableInfo(model));
//这个类有一个构造方法 接收继承Model的类  
TableInfo其实就是同来  获取@Table @Column注解的值
public TableInfo(Class<? extends Model> type) 
		
		//这段代码其实就是 先获取@Table注解的这个class文件
		final Table tableAnnotation = type.getAnnotation(Table.class);
		//如果不为空  就把这个文件的name值和id值取出来
        if (tableAnnotation != null) {
			mTableName = tableAnnotation.name();
			mIdName = tableAnnotation.id();
		}
		//如果为null 也就是别人在使用的时候 没有进行name字段的赋值
		就使用这个类的名字来当 这个name 的值
		else {
			mTableName = type.getSimpleName();
        }
		//最后把 id存放在Map集合中
		mColumnNames.put(idField, mIdName);
	

		获取Table注解后   下面获取@Column的属性
			获取自定义所有的字段 存入到集合中
	 List<Field> fields = new LinkedList<Field>(ReflectionUtils.getDeclaredColumnFields(type));
		循环集合
		 for (Field field : fields) {
			这句话的意思就是 如果字段引用了Column的注解 返回true
            if (field.isAnnotationPresent(Column.class)) {
					//返回Column注解的Class文件
                final Column columnAnnotation = field.getAnnotation(Column.class);
					//把name值取出来 如果没有就按照 字段的名字
                String columnName = columnAnnotation.name();
                if (TextUtils.isEmpty(columnName)) {
                    columnName = field.getName();
                }
				最后放在一个集合中  key就是field，value是这个name的值
                mColumnNames.put(field, columnName);
			也就是说mColumnNames这个Map集合中就是存放创建数据库表的那些个列名 从id开始 比如name  age 等等
            }
        }	

DatabaseHelper = new DatabaseHelper(configuration)

	继承了SQLiteOpenHelper
	构造函数 获取之前 Configuration中获取到的 版本和数据库名字	
	在它的onCreate中 调用了executeCreate(db);
	这个方法 就是在循环  有几个tableInfo  也就是有几张表
	
	for (TableInfo tableInfo : Cache.getTableInfos()) {
				db.execSQL(SQLiteUtils.createTableDefinition(tableInfo));
			}
	然后执行createTableDefinition方法
		里面就是循环么一个Tableinfo类中那个集合
			也就是每个表中 的所有字段
	for (Field field : tableInfo.getFields()) {
			String definition = createColumnDefinition(tableInfo, field);
	最后 return String.format("CREATE TABLE IF NOT EXISTS %s (%s);", tableInfo.getTableName(),
	TextUtils.join(", ", definitions));
	 就是把表名取出来  替换第一个占位符
	TextUtils.join(", ", definitions))这句话就是 获取这个列名的名字 类型
	和约束条件等
	通过createColumnDefinition（TableInfo tableInfo, Field field）

	这个方法 返回一个 String 在里面判断获取 类型和一些自定义的约束
	field.getType();获取类型
	@column(name="",unique="",notNull="")等
	获取完以后 就拼接成一句完整的 创建表的 sql语句
		
		





1. 程序入口 
 
```java

	public static void initialize(Context context) {
		initialize(new Configuration.Builder(context).create());
	}

	public static void initialize(Configuration configuration) {
		initialize(configuration, false);
	}

	public static void initialize(Context context, boolean loggingEnabled) {
		initialize(new Configuration.Builder(context).create(), loggingEnabled);
	}

	public static void initialize(Configuration configuration, boolean loggingEnabled) {
		// Set logging enabled first
		setLoggingEnabled(loggingEnabled);
		Cache.initialize(configuration);
	}

 ```

2. 分析Configuration

create方法主要从清单文件里面获取 数据库等信息。
   ```java
public Configuration create() {
			Configuration configuration = new Configuration(mContext);
			configuration.mCacheSize = mCacheSize;

			// Get database name from meta-data
			// 获取数据库名字
			if (mDatabaseName != null) {
				configuration.mDatabaseName = mDatabaseName;
			} else {
				configuration.mDatabaseName = getMetaDataDatabaseNameOrDefault();
			}
			// 获取数据库版本号
			// Get database version from meta-data
			if (mDatabaseVersion != null) {
				configuration.mDatabaseVersion = mDatabaseVersion;
			} else {
				configuration.mDatabaseVersion = getMetaDataDatabaseVersionOrDefault();
			}

			// Get SQL parser from meta-data
			if (mSqlParser != null) {
			    configuration.mSqlParser = mSqlParser;
			} else {
			    configuration.mSqlParser = getMetaDataSqlParserOrDefault();
			}
			
			// Get model classes from meta-data
			// 获取清单文件里面配置的models
			if (mModelClasses != null) {
				configuration.mModelClasses = mModelClasses;
			} else {
				final String modelList = ReflectionUtils.getMetaData(mContext, AA_MODELS);
				if (modelList != null) {
					configuration.mModelClasses = loadModelList(modelList.split(","));
				}
			}

			// Get type serializer classes from meta-data
			if (mTypeSerializers != null) {
				configuration.mTypeSerializers = mTypeSerializers;
			} else {
				final String serializerList = ReflectionUtils.getMetaData(mContext, AA_SERIALIZERS);
				if (serializerList != null) {
					configuration.mTypeSerializers = loadSerializerList(serializerList.split(","));
				}
			}

			return configuration;
		}


	private String getMetaDataDatabaseNameOrDefault() {
			String aaName = ReflectionUtils.getMetaData(mContext, AA_DB_NAME);
			if (aaName == null) {
				aaName = DEFAULT_DB_NAME;
			}

			return aaName;
		}


	@SuppressWarnings("unchecked")
	public static <T> T getMetaData(Context context, String name) {
		try {
			final ApplicationInfo ai = context.getPackageManager().getApplicationInfo(context.getPackageName(),
					PackageManager.GET_META_DATA);

			if (ai.metaData != null) {
				return (T) ai.metaData.get(name);
			}
		}
		catch (Exception e) {
			Log.w("Couldn't find meta-data: " + name);
		}

		return null;
	}


   ```

### 3. 初始化一个Cache

	public static synchronized void initialize(Configuration configuration) {
		if (sIsInitialized) {
			Log.v("ActiveAndroid already initialized.");
			return;
		}

		sContext = configuration.getContext();
		sModelInfo = new ModelInfo(configuration);
		sDatabaseHelper = new DatabaseHelper(configuration);

		// TODO: It would be nice to override sizeOf here and calculate the memory
		// actually used, however at this point it seems like the reflection
		// required would be too costly to be of any benefit. We'll just set a max
		// object size instead.
		sEntities = new LruCache<String, Model>(configuration.getCacheSize());

		openDatabase();

		sIsInitialized = true;

		Log.v("ActiveAndroid initialized successfully.");
	}


### ModelInfo

	public ModelInfo(Configuration configuration) {
			if (!loadModelFromMetaData(configuration)) {
				try {
					scanForModel(configuration.getContext());
				}
				catch (IOException e) {
					Log.e("Couldn't open source path.", e);
				}
			}
	
			Log.i("ModelInfo loaded.");
		}

	private boolean loadModelFromMetaData(Configuration configuration) {
			if (!configuration.isValid()) {
				return false;
			}

		final List<Class<? extends Model>> models = configuration.getModelClasses();
		if (models != null) {
			for (Class<? extends Model> model : models) {
				mTableInfos.put(model, new TableInfo(model));
			}
		}

		final List<Class<? extends TypeSerializer>> typeSerializers = configuration.getTypeSerializers();
		if (typeSerializers != null) {
			for (Class<? extends TypeSerializer> typeSerializer : typeSerializers) {
				try {
					TypeSerializer instance = typeSerializer.newInstance();
					mTypeSerializers.put(instance.getDeserializedType(), instance);
				}
				catch (InstantiationException e) {
					Log.e("Couldn't instantiate TypeSerializer.", e);
				}
				catch (IllegalAccessException e) {
					Log.e("IllegalAccessException", e);
				}
			}
		}

		return true;
	}


	private void scanForModel(Context context) throws IOException {
			String packageName = context.getPackageName();
			String sourcePath = context.getApplicationInfo().sourceDir;
			List<String> paths = new ArrayList<String>();

		if (sourcePath != null && !(new File(sourcePath).isDirectory())) {
			DexFile dexfile = new DexFile(sourcePath);
			Enumeration<String> entries = dexfile.entries();

			while (entries.hasMoreElements()) {
				paths.add(entries.nextElement());
			}
		}
		// Robolectric fallback
		else {
			ClassLoader classLoader = Thread.currentThread().getContextClassLoader();
			Enumeration<URL> resources = classLoader.getResources("");

			while (resources.hasMoreElements()) {
				String path = resources.nextElement().getFile();
				if (path.contains("bin") || path.contains("classes")) {
					paths.add(path);
				}
			}
		}

		for (String path : paths) {
			File file = new File(path);
			scanForModelClasses(file, packageName, context.getClassLoader());
		}
	}

---

	// 存储model的信息
	// TableInfo 主要是用来封装Model的表信息
	
	// 表名 ，列名 
	
	<Person.class,
	private Map<Class<? extends Model>, TableInfo> mTableInfos = new HashMap<Class<? extends Model>, TableInfo>();
	
	@Table(name="Person",Id= _id)
	public class Person extends Model{
	
	}
	
	public TableInfo(Class<? extends Model> type) {
			mType = type;
	// 获取注解对象 
			final Table tableAnnotation = type.getAnnotation(Table.class);
	
	        if (tableAnnotation != null) {
	// 表名
				mTableName = tableAnnotation.name();
	// id的名字
				mIdName = tableAnnotation.id();
			}
			else {
	//
				mTableName = type.getSimpleName();
	        }
	
	        // Manually add the id column since it is not declared like the other columns.
	        Field idField = getIdField(type);
	        mColumnNames.put(idField, mIdName);
	
	        List<Field> fields = new LinkedList<Field>(ReflectionUtils.getDeclaredColumnFields(type));
	        Collections.reverse(fields);
	
	        for (Field field : fields) {
	            if (field.isAnnotationPresent(Column.class)) {
	                final Column columnAnnotation = field.getAnnotation(Column.class);
	                String columnName = columnAnnotation.name();
	                if (TextUtils.isEmpty(columnName)) {
	                    columnName = field.getName();
	                }
	
	                mColumnNames.put(field, columnName);
	            }
	        }
	
		}

