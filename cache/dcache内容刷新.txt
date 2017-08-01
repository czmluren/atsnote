dcache内容刷新

URL刷新原理:
使用的是ATS提供的HTTP_UI接口，直接循环调用 CacheProcessor::remove 删除URL。

URL刷新触发操作:
ShowCache::invalidate_url //通过 if (strcmp(show_cache_urlstrs[urlstrs_index], "") == 0){ return complete(event, e); } 结束循环。 
ShowCache::handleCacheInvalidateComplete  //调用 ShowCache::invalidate_url
CacheProcessor::remove //删除url，并回调 ShowCache::handleCacheInvalidateComplete



目录刷新原理:
首先接收目录刷新请求，根据host和path来更新HASH表，后面每个请求都会通过HASH表去判断该请求是否被目录刷新标记过，再来判断是否过期。
HASH表里面的数据，在接收到目录刷新请求后会自动更新 invalidate.config（ATS启动时使用）。

目录刷新初始化:
main
CacheRefreshProcessor::start
RefreshHandler::start
initCacheInvalidate  //读取 invalidate.config 配置文件，并生成 CacheInvalidate::path_ht 的HASH表，以 host 和 path 为维度。
    CacheInvalidate::CacheInvalidate
    CacheInvalidate::BuildTable
    CacheInvalidate::BuildTableFromFile
thread->schedule_every(refreshHandler, REFRESH_PERIOD);  //RefreshHandler::mainEvent 循环执行，间隔时间20毫秒

目录刷新操作:
ShowCache::invalidate_path
addInvalidatePath
CacheRefreshProcessor::NewEntry
RefreshEntry::init //根据host和path生成事件RefreshEntry元素 
RefreshEntry::mainEvent  //回调CacheRefreshProcessor::push_entry
CacheRefreshProcessor::push_entry  //将RefreshEntry元素添加到 RefreshHandler::entries 队列中
RefreshHandler::mainEvent //定时消费 RefreshHandler::entries 队列中的事件
CacheInvalidate::NewEntry //将 RefreshEntry 事件的目录刷新操作添加到 CacheInvalidate::path_ht 的HASH表中  
eventProcessor.schedule_in(NEW(new CCI_UpdateContinuation(invalidate_mutex)), CACHE_INVALIDATE_TIMEOUT, ET_CACHE);
    CCI_UpdateContinuation::file_update_handler  //60秒后更新 invalidate.config 配置文件
    CacheInvalidate::WriteCacheInvalidateFile  //将 CacheInvalidate::path_ht 的HASH表中的数据更新到 invalidate.config 配置文件中

目录刷新触发操作:
HttpTransact::HandleRequest
update_cache_control_information_from_config
getCacheControl
getCacheInvalidate
CacheInvalidate::Match //根据HASH表更新请求的 result->invalidate_after 参数
HttpTransact::what_is_document_freshness //根据 result->invalidate_after 参数来判断请求是否过期

