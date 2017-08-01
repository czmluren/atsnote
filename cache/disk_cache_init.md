ATS磁盘读取初始化流程：
CacheProcessor::start 

CacheProcessor::start_internal

    //<Main.cc:1900 (main)> traffic server running

CacheDisk::open    //一个磁盘调用一次
    SET_HANDLER(&CacheDisk::openStart);

CacheDisk::openStart
    SET_HANDLER(&CacheDisk::openDone);

CacheDisk::openDone
    SET_HANDLER(&CacheDisk::syncDone);
    eventProcessor.schedule_in(this, HRTIME_MSECONDS(5), ET_CALL);

CacheProcessor::diskInitialized   //直到open最后一个磁盘才执行 Cache::open 函数

Cache::open  //只执行一次
    eventProcessor.schedule_imm_signal(NEW (new VolInit(cp->vols[vol_no], d->path, blocks, q->b->offset, vol_clear, cp->vol_number))); //一个磁盘调用一次

VolInit::mainEvent

Vol::init  //NOTE: reading directory '/dev/sdb 368640:244181087'
    vol_dirlen //directory的长度计算
    SET_HANDLER(&Vol::handle_header_read);  
    eventProcessor.schedule_imm(this, ET_CALL);

Vol::handle_header_read  //NOTE: using directory A/B for '/dev/sdb 368640:244181087'
    SET_HANDLER(&Vol::handle_dir_read);

Vol::handle_dir_read

Vol::recover_data
    SET_HANDLER(&Vol::handle_recover_from_data);

Vol::handle_recover_from_data  //NOTE: recovery clearing offsets [2647206400, 2655595008] sync_serial 12 next 13 ,该磁盘上存储过数据才会打印此信息
    SET_HANDLER(&Vol::handle_recover_write_dir);

Vol::handle_recover_write_dir
    SET_HANDLER(&Vol::dir_init_done);

Vol::dir_init_done
    SET_HANDLER(&Vol::aggWrite);

Cache::vol_initialized  //直到初始化最后一个磁盘才执行 Cache::open_done 函数

Cache::open_done  //只执行一次

CacheHostTable::CacheHostTable

CacheHostTable::BuildTable

CacheHostTable::BuildTableFromString

CacheHostRecord::Init

build_vol_hash_table  // (cache_init) build_vol_hash_table 0 request 2726 got 2746

CacheProcessor::cacheInitialized   //<Cache.cc:1071 (cacheInitialized)> cache enabled

