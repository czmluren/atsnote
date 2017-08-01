基于Dcache版本研究

1、DNSHandler
DNSHandler类负责读取请求，发送DNS查询，并处理响应，然后调用上层应用进行响应处理：
struct DNSHandler: public Continuation
    DNSConnection con[MAX_NAMED];
    Queue<DNSEntry> entries;
    Queue<DNSConnection> triggered;

2、函数调用
main --> DNSProcessor.start --> DNSProcessor.open --> DNSHandler.startEvent --> DNSHandler.mainEvent
ATS启动时调用DNSProcessor.start，读取配置参数，进行初始化，然后调用DNSProcessor.open建立与DNS之间的连接
 
3、int DNSHandler::mainEvent(int event, Event *e)
读取 triggered 队列中的响应，发送 entries 队列中的请求
    recv_dns(event, e); //接收响应
    write_dns(this);  //发送请求

4、int  DNSEntry::mainEvent(int event, Event *e)
DNSHandler::mainEvent 中处理的请求是在进行DNS查询时，由 DNSEntry::mainEvent 将DNS查询请求加入 DNSHandler 的 entries 队列：
    dnsH->entries.enqueue(this);
    write_dns(dnsH);
函数调用关系：
DNSEntry.mainEvent 52
    DNSEntry.init 
        DNSProcessor.getby 
            DNSProcessor.getSRVbyname 
            DNSProcessor.gethostbyname 
            DNSProcessor.gethostbyaddr 
    DNSEntry.delayEvent  

5、DNS响应处理过程：
DNSHandler 收到响应，调用 DNSEntry.postEvent 将响应传给上层应用，上层应用调用 HostDBContinuation::dnsEvent 进行处理：
main() ---> DNSProcessor::start() ---> DNSProcessor::open() ---> DNSHandler::startEvent() ---> DNSHandler::mainEvent() ---> DNSHandler::recv_dns()
 ---> dns_process() ---> dns_result() ---> DNSEntry::postEvent() ---> HostDBContinuation::dnsEvent() ---> HttpSM::state_hostdb_lookup() ---> HttpSM::process_hostdb_info()

DNSHandler.mainEvent --> DNSHandler.recv_dns --> dns_process --> dns_result --> DNSEntry.postEvent


