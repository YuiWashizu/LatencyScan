# LatencyScan
YARRに実装されていない、LatencyScanを一応できるようにする手順

## 前提
* ExtTriggerScanできるようなFWであること
* RD53AとFPGAボードが、データ通信用のDP1とExtTrigger取得用のDP2でつながっていること

## 準備
1. 以下のコマンドを実行
```
% git clone https://gitlab.cern.ch/YARR/YARR.git Yarr
% cd Yarr
% git checkout ff468636236a087d974fe2400eb1797e30d211d6
% cd src
% git clone xxxx
% mv ./LatencyScan/std_latencyscan.json ./configs/scans/rd53a/
```
1. `libYarr/StdDataGatherer.cpp`に以下の**追加**と書かれた部分を追加
```
void StdDataGatherer::execPart2()  {
  int past_eve = 0; //追加
  int past_eve_fin = 100; //1つのLatency値あたり、取得したいイベント数
  while (done == 0) {
	  std::unique_ptr<RawDataContainer> rdc(new RawDataContainer());
    rate = g_rx->getDataRate();
	  if (verbose)
            std::cout << " --> Data Rate: " << rate/256.0/1024.0 << " MB/s" << std::endl;
    done = g_tx->isTrigDone();
    do {
      newData =  g_rx->readData();
      if (newData != NULL) {
        rdc->add(newData);
        count += newData->words;
        newData = NULL;
        past_eve ++;
      }
      std::this_thread::sleep_for(std::chrono::microseconds(100));
   } while (newData != NULL && signaled == 0 && !killswitch && past_eve < past_eve_fin); //追加
   if (newData != NULL)
    delete newData;
   rdc->stat = *g_stat;
   storage->pushData(std::move(rdc));
   if (signaled == 1 || killswitch || past_eve == past_eve_fin) { //追加
   .......
```
1. コンパイルする
```
% cd src
% make -j4
```
## 実行
以下のコマンドを入力すれば、Latency値の分布を得られる。
```
% bin/scanConsole -r configs/controller/specCfgExtTrigger.json -c configs/connectivity/example_rd53a_setup.json -s configs/scans/rd53a/std_latencyscan.json
 <Some Text>
% ./convertRaw.sh ./data/00000_std_latencyscan/JohnDoe_data.raw
% ./setup.sh
% root -l
[0] .L fromHitTree.h
[1] .x LatencyAnalysis.C+("./data/00000_std_latencyscan/JohnDoe_data.root")
```
