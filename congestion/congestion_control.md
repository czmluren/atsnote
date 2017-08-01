基于ATS6.1.1代码研究

变量和函数解释
t_state 中与拥塞控制相关的参数
// congestion control
CongestionEntry *pCongestionEntry;
StateMachineAction_t congest_saved_next_action; //下一状态保存，有 HttpTransact::STATE_MACHINE_ACTION_UNDEFINED, HttpTransact::ORIGIN_SERVER_OPEN 和 
                                                // HttpTransact::ORIGIN_SERVER_RAW_OPEN 这三个值，主要用于在拥塞状态下的状态跳转
int congestion_control_crat;        // 'client retry after',这个主要是影响访问日志的记录
int congestion_congested_or_failed;  //表示此请求是否拥塞或失败，主要用于对 t_state.pCongestionEntry->go_alive(); 即 m_congested 状态更新为 0
int congestion_connection_opened;   //表示此请求的拥塞控制功能是否打开，主要用于 pCongestionEntry->connection_closed(); 即 m_num_connections 值的更新

FailHistory 类 
成员变量
long start;  //统计周期内第一个事件的访问时间

int bin_len;  //其值为 (fail_window + CONG_HIST_ENTRIES) / CONG_HIST_ENTRIES ，即 bins[i] 失败事件的统计间隔时间
int length;  //其值为 bin_len * CONG_HIST_ENTRIES ，即所有环形数组bins元素的间隔时间之和

cong_hist_t bins[CONG_HIST_ENTRIES];  //环形数组，bins[i]元素值对应着每个间隔时间的失败事件数
int cur_index;   //在环形数组bins的起始位置 

notes: 所有失败事件的次数和间隔时间通过一个环形数组 bins 保存着，每个 bins[i] 元素对应着 bin_len 这段时间内失败的事件数

long last_event;  //上次最近失败事件注册时间
int events;  //当前间隔内失败的事件总个数

成员函数被调顺序
HttpTransact::handle_parent_died --> CongestionEntry::failed_at --> FailHistory::regist_event --> FailHistory::init_event 
CongestionEntry::CongestionEntry --> CongestionEntry::clearFailHistory --> FailHistory::init
CongestionControlEnabledChanged --> revalidateCongestionDB --> CongestionDB::revalidateBucket --> CongestionEntry::validate --> CongestionEntry::applyNewRule --> CongestionEntry::init --> CongestionEntry::clearFailHistory --> FailHistory::init

notes: 此类通过 max_connection_failures 和 fail_window 参数来控制拥塞

CongestionEntry 类
成员变量
// State -- connection failures
FailHistory m_history;  //保存 CongestionEntry 一定时间内访问源站失败的记录
ink_hrtime m_last_congested;  //上次失败事件的访问时间，由 m_last_congested = m_history.last_event; 赋值，用于日志打印，基本无用
volatile int m_congested;     //0 | 1  当前拥塞判断值，主要是与 m_history 相关的拥塞开关
int m_stat_congested_conn_failures; //通过 m_congested 开关拥塞失败连接的总次数

volatile int m_M_congested; //连接数超过限制时的拥塞开关
ink_hrtime m_last_M_congested;  //上次连接数超过限制时被拥塞的时间

// State -- concorrent connections
int m_num_connections;  //当前连接源站的总连接数
int m_stat_congested_max_conn; //与 m_M_congested 开关相关，连接数超过限制时被限制的总连接数

// Reference count
int m_ref_count;  //所有请求通过引用计数器对 CongestionEntry 对象共享的，只有在计数器值未0时才会删除此对象

notes: m_congested 的放开是受 m_congested, m_M_congested 和源站是否挂掉了一起控制的

成员函数
virtual RD_Type data_type(void);   //主要用于配置项查找 CongestionControlRecord::UpdateMatch
inline bool proxy_retry(ink_hrtime t);  //通过当前时间、 m_history.last_event 、 proxy_retry_interval 来判断是否可以重试
inline int client_retry_after();  //返给客户端时让其重试的时间，其值是 proxy_retry_interval, m_history.last_event, 当前时间, client_wait_interval, wait_interval_alpha 一起计算出
inline int connect_timeout();  //通过判断 m_congested 返回 dead_os_conn_timeout 或 live_os_conn_timeout
inline int connect_retries();  //通过判断 m_congested 返回 dead_os_conn_retries 或 live_os_conn_retries
void stat_inc_F(); //m_stat_congested_conn_failures ,被统计的地方有 HttpSM::do_http_server_open 和 HttpTransact::handle_server_died
void stat_inc_M(); //m_stat_congested_max_conn ,被统计的地方只有 HttpSM::do_http_server_open

congestion.config 配置初始化
main (argv=0x7fffffffe618) at Main.cc:1550
initCongestionControl () at Congestion.cc:390
CongestionMatcherTable::reconfigure () at Congestion.cc:405
CongestionMatcherTable::CongestionMatcherTable (this=0x1086050, file_var=0x70d768 "proxy.config.http.congestion_control.filename", name=0x70d1b1 "[CongestionControl]", tags=0x70d920) at Congestion.cc:81
ControlMatcher<CongestionControlRecord, CongestionControlRule>::ControlMatcher (this=0x1086060, file_var=0x70d768 "proxy.config.http.congestion_control.filename", name=0x70d1b1 "[CongestionControl]", tags=0x70d920, flags_in=31) at ControlMatcher.cc:724
ControlMatcher<CongestionControlRecord, CongestionControlRule>::BuildTable (this=0x1086060) at ControlMatcher.cc:974
ControlMatcher<CongestionControlRecord, CongestionControlRule>::BuildTableFromString (this=0x1086060, file_buf=0x1085250 "#") at ControlMatcher.cc:925
HostMatcher<CongestionControlRecord, CongestionControlRule>::NewEntry (this=0x1083b80, line_info=0x10843d0) at ControlMatcher.cc:211
CongestionControlRecord::Init (this=0x10846d8, line_info=0x10843d0) at Congestion.cc:186


CongestionTest.cc 这个文件不用看，ATS没有用到


数据结构技巧点：
配置读取模板类：
1. ControlMatcher 类是用于读取配置的模板类，里面封装了 HostMatcher, IpMatcher, RegexMatcher, UrlMatcher, HostRegexMatcher 模板类成员
   分别用来对配置文件的 (dest_domain, dest_host), dest_ip, url_regex, url, host_regex 所指定的配置项进行解析和存储
2. ControlMatcher::BuildTable 使用 readIntoBuffer 函数一次将配置文件的所有内容 file_buf 内存中,再使用 ControlMatcher::BuildTableFromString 来进行解析，
   不同配置所支持的 primary_destination 选项不同，主要是该继承类的 config_tags 结构体决定的
3. ControlMatcher::BuildTableFromString 主要计算出配置文件中 primary_destination 不同选项的是否配置，计算不同配置项的有效行数，
   并解析出来的有效配置行存储到 matcher_line *first 单链表中，将不同的配置存储到 ControlMatcher 对应的成员变量中
4. RegexMatcher<Data, Result>, UrlMatcher<Data, Result>,...等类，配置文件的配置项是存储到 Data 类中的，存储空间是通过 [*]Matcher::AllocateSpace 函数申请分配的，
   其中 <Data, Result> 是由 ControlMatcher<Data, Result> 中 Data和Result 传进去的，申请分配时实质是 new 的 Data 类，Data类数据都是存储到 data_array 变量中。
5. ControlMatcher::BuildTableFromString 初步解析出来的配置 matcher_line *first 是通过 RegexMatcher, UrlMatcher 等类的 NewEntry 函数存入， 
   NewEntry 调 Data 中的 Init 函数来对存储结构进行初始化和存入数据
6. ControlMatcher::Match 是通过HTTP请求来匹配出对应的配置项规则，查找不同类型的配置项有先后顺序，在一类型的配置项未查找到再查找另外一类，其查找顺序参考此函数

配置读取存储和初始化：
1. CongestionControlRecord 此类的一个对象对应配置文件的一条配置项
2. CongestionControlRule(Result) 此类作为 CongestionControlRecord(Data) 指针存储容器，将查找的 Data 用 Result 来返回
3. CongestionMatcherTable 类是继承于 ControlMatcher<CongestionControlRecord, CongestionControlRule> ,配置文件的读取和解析是在 CongestionMatcherTable 构造函数中完成的
   CongestionMatcherTable 类的实例化是 CongestionMatcherTable::reconfigure 函数中完成，并将该类对象直接赋给文件静态 CongestionMatcher 指针，即此指针存储着配置文件的所有配置
4. CongestionMatcherTable::reconfigure 函数还调用了 revalidateCongestionDB 函数，对 theCongestionDB 指针进行了类分配，并对其进行初始化
5. HTTP请求查找配置规则是通过 CongestionControlled 函数查找的

HASH存储模板类：
1. 模板类 HashTableEntry<key_t, data_t> 是一个单链表类结点，里面的 data 成员即 data_t 类型
2. 模板类 IMTHashTable<key_t, data_t> 的数据成员 buckets[] 是一个 HashTableEntry 类的数组，每个数组元素中存一个 HashTableEntry 类的单链表
3. 模板类 IMTHashTable 的HASH表是使用拉链法来解决冲突的
4. 数据存入 IMTHashTable 中 buckets[] 中时，通过 IMTHashTable::bucket_id 函数将 key 和 bucket_num 转换了一次，以保证存入对应的 buckets[] 的某个单链表中；
5. 模板类 IMTHashTable::resize 方法是重新分配一定空间的HASH桶，size即 new_buckets[] 数组的最大数组元素个数，先将旧 buckets[] 中的所有元素通过头插法放入到 new_buckets[] 中，
   再将旧 buckets[] 删除，用新 new_buckets[] 将其替换
6. IMTHashTable::insert_entry 先找了以前是否存入过，存过则用新 data_t 替换，并返回旧数据 data_t ，方便释放；
   未存入过，则先分配一个 HashTableEntry 用来保存新 data_t 数据，并使用头插法插入到 buckets[] 的某个单链表中；
   插入时，是否需要用 IMTHashTable::resize 进行扩容，是通过 HASH 链表平均长度 MT_HASHTABLE_MAX_CHAIN_AVG_LEN 来决定的，
   扩容前会利用 IMTHashTable::GC 通过传入的 m_gc_func 和 m_pre_gc_func 函数指针判断删掉一部分过期数据
7. IMTHashTable 类中的 first_entry,next_entry,cur_entry,remove_entry 方法是利用 HashTableIteratorState 类在 buckets[] 的某个单链表来实现迭代器功能
8. MTHashTable<key_t, data_t> 是利用锁对 IMTHashTable<key_t, data_t> 的数据成员和方法进行的一次封装, hashTables[] 中存的是 IMTHashTable 类数组, 
   hashTables[] 的数组长度和 locks 的个数都是 MT_HASHTABLE_PARTITIONS ,hashTables[] 数组的每个元素都对应一把锁 lock,两者一一对应。
9. MTHashTable 是个二级HASH桶
   第一级是 MTHashTable 的 *hashTables[MT_HASHTABLE_PARTITIONS] ，此数组的最大元素个数一直是 MT_HASHTABLE_PARTITIONS ，不会发生变化；
   第二级是 IMTHashTable 的 *buckets[]，会根据 MT_HASHTABLE_MAX_CHAIN_AVG_LEN 来自动判断 *buckets[] 是否需要扩容，以保证此桶的单链表的长度不至于过长，提高查找速度。

数据存储类：
1. FailHistory 类存储该域名失败历史记录
2. CongestionEntry 继承于 RequestData 类，里面存储着每个域名的拥塞状态信息和历史记录，与 CongestionControlRecord 配置是多对一的关系
3. 每个 CongestionEntry 类对象都有一把 Ptr<ProxyMutex> m_hist_lock 锁方便该域名对失败历史记录 FailHistory m_history 的多线程操作，其他的变量是使用的原子操作
4. CongestRequestParam 类是一个双链表类结点，里面的数据成员是 CongestionEntry 和与其相关的一些操作标识
5. CongestionDB 类是对 MTHashTable<uint64_t, CongestionEntry *> 类的继承，是 CongestionEntry 类对象的存储容器，元素存在 CongestionDB *theCongestionDB 所指向的类对象中
6. 存入 MTHashTable 的 key 是通过 make_key 来计算的，可以使用 host, ip, prefix, CongestionControlRecord 等通过md5计算出key ，每个 CongestionEntry 对象都有一个 key 与之对应
7. CongestionDB 类的 *todo_lists 数组链表是当拿取 MTHashTable 的桶锁 lock 失败后，此时数据成员 CongestRequestParam 存放的地方，todo_lists 数组个数与 hashTables[] 个数相同，
   都是 MT_HASHTABLE_PARTITIONS ，后续获取对应的 MTHashTable lock 后，再通过调用 CongestionDB::RunTodoList 函数对其分类处理，放置到 MTHashTable 的HASH桶 hashTables[] 中 
   *todo_lists 的链表操作是使用的线性安全的链表函数 ink_atomiclist_init, ink_atomiclist_push, ink_atomiclist_popall
8. CongestionDB 类初始化时，是将 congestEntryGC 和 preCongestEntryGC 的函数指针分别传给 IMTHashTable 类的 m_gc_func 和 m_pre_gc_func 来删除过期数据和扩容HASH表，
   其中 CongestionEntry 就是存储在 IMTHashTable 的数据成员 HashTableEntry<key_t, data_t> *buckets[] 的单链表结点中，结点 HashTableEntry 数据成员 data_t 即 CongestionEntry ，
   theCongestionDB 中的 CongestionEntry 一般不删除，只有 HASH 表需要扩容前才会调用 IMTHashTable::GC 通过 m_gc_func(congestEntryGC) 将其删除
9. CongestionDBCont 类继承于 Continuation 类，主要是用于查找 theCongestionDB 的 CongestionEntry 时拿锁失败后，用于事件调度用的 eventProcessor.schedule_in ，实现异步避免阻塞
10.revalidateCongestionDB 函数是初始化和热加载时，会重新更新 theCongestionDB ，删除失效 CongestionEntry ，并对所有 CongestionEntry 应用新的规则 CongestionControlRecord

查找、存储和拥塞调用流程：
1. get_congest_entry 函数先通过 HTTP请求用 CongestionControlled 找出对应的规则 CongestionControlRecord ，再用 make_key 来计算出 key ，
   再在 theCongestionDB 中找出对应的 CongestionEntry ，若拿锁失败则使用调度隔断时间进行重试，若再次失败则继续重试直到拿锁成功或 m_action 取消为止
2. key 的计算是通过 源站域名或IP、端口port、前缀prefix（URL的前缀） 一起计算而出，详细可以参考 make_key 函数
3. 拥塞功能开启配置必须满足下面两点：
   I>  proxy.config.http.congestion_control.enabled 必须开启
   II> congestion.config 配置必须配置，且请求能够匹配上，未匹配上的就没有拥塞功能
4. t_state.pCongestionEntry 赋值在 HttpSM::do_congestion_control_lookup 函数中，此函数在 HttpSM::set_next_state 中的 HttpTransact::ORIGIN_SERVER_OPEN 和 
   HttpTransact::ORIGIN_SERVER_RAW_OPEN 这两处状态下调用
5. 拥塞功能主要针对两类： 
   I> m_congested 失败重试限制的  
   II> m_M_congested 最大连接数限制的
6. m_congested 和 m_M_congested 的参数设置和使用
   m_congested:
      //set 
      CongestionEntry::go_alive   <-- HttpSM::kill_this
      //use & set 
      CongestionEntry::failed_at      <-- HttpTransact::handle_server_died
      CongestionEntry::compCongested  <-- CongestionEntry::failed_at
      //use
      CongestionEntry::usefulInfo   //扩容函数指针调用 congestEntryGC
      CongestionEntry::F_congested  <-- CongestionEntry::client_retry_after  <-- HttpTransact::build_error_response
                                    <-- CongestionEntry::congested           <-- get_congest_list ()
                                    <-- CongestionEntry::connect_retries     <-- HttpTransact::handle_response_from_server
                                    <-- CongestionEntry::connect_timeout     <-- HttpSM::do_http_server_open
                                                                             <-- HttpSM::attach_server_session
                                    <-- HttpSM::do_http_server_open
                                    <-- HttpTransact::handle_server_died
   m_M_congested:
      //set & use --by m_num_connections & pRecord->max_connection  
      CongestionEntry::M_congested <-- HttpSM::do_http_server_open 
      CongestionEntry::connection_closed  <-- HttpTransact::State::destroy
      //use
      CongestionEntry::congested  <-- get_congest_list(影响此函数的日志打印)
  
   notes: 
   HttpSM::do_http_server_open 通过 CONGESTION_EVENT_CONGESTED_ON_F 和 CONGESTION_EVENT_CONGESTED_ON_M 状态值，影响下面两个函数
   HttpSM::state_raw_http_server_open 和 HttpSM::state_http_server_open 的 HttpTransact::CONGEST_CONTROL_CONGESTED_ON_F 和 
   HttpTransact::CONGEST_CONTROL_CONGESTED_ON_M 状态跳转

   I、  失败事件的拥塞功能 最终是通过设置ATS本身自带重试功能参数来实现的，retry_after(MIME_FIELD_RETRY_AFTER), max_connect_retries, connect_timeout
   II、 通过连接数拥塞的请求，不会立马返回失败结果给客户端，而是通过 CONGESTION_EVENT_CONGESTED_ON_F 和 CONGESTION_EVENT_CONGESTED_ON_M 事件重试连接源站，
   多次失败后，返回结果给客户端让其进行重试
   III、 最终的核心还是ATS的超时和重试
