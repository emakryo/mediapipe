---
id: tvr4j
title: Framework walkthrough
file_version: 1.1.2
app_version: 1.5.5
---

# 概要

`📄 mediapipe/framework`以下にあるmediapipeのパイプライン処理の実装に関して説明します。

*   CalculatorGraphのインターフェース

*   CalculatorGraphのデータ構造

*   Packetの入力・実行・出力まで

    *   グラフレベルの入力からタスクのスケジューリングまで

    *   ソースノードのスケジューリング

    *   スケジュールされたタスクの実行

    *   出力ストリームに対するコールバックが呼ばれるまで

# CalculatorGraphのインターフェース

`CalculatorGraph`<swm-token data-swm-token=":mediapipe/framework/calculator_graph.h:96:2:2:`class CalculatorGraph {`"/>がアプリケーションから利用されるエントリーポイントになります。主に使われるメソッドをいくつか挙げます。

## コンストラクタ

<br/>

`CalculatorGraphConfig`<swm-token data-swm-token=":mediapipe/framework/calculator.proto:224:2:2:`message CalculatorGraphConfig {`"/> で定義された設定から初期化
<!-- NOTE-swimm-snippet: the lines below link your snippet to Swimm -->
### 📄 mediapipe/framework/calculator_graph.h
```c
120      // Initializes the graph from its proto description (using Initialize())
121      // and crashes if something goes wrong.
122      explicit CalculatorGraph(CalculatorGraphConfig config);
123      virtual ~CalculatorGraph();
124    
```

<br/>

## 実行開始

<br/>

非同期に実行を開始する。（同期的に実行終了まで待つメソッド `Run`<swm-token data-swm-token=":mediapipe/framework/calculator_graph.h:186:5:5:`  absl::Status Run() { return Run({}); }`"/> も存在する）
<!-- NOTE-swimm-snippet: the lines below link your snippet to Swimm -->
### 📄 mediapipe/framework/calculator_graph.h
```c
188      // Start a run of the graph.  StartRun, WaitUntilDone, Cancel, HasError,
189      // AddPacketToInputStream, and CloseInputStream allow more control over
190      // the execution of the graph run.  You can insert packets directly into
191      // a stream while the graph is running. Once StartRun has been called,
192      // the graph will continue to run until all work is either done or canceled,
193      // meaning that either WaitUntilDone() or Cancel() has been called and has
194      // completed. If StartRun returns an error, then the graph is not started and
195      // a subsequent call to StartRun can be attempted.
196      //
197      // Example:
198      //   MP_RETURN_IF_ERROR(graph.StartRun(...));
199      //   while (true) {
200      //     if (graph.HasError() || want_to_stop) break;
201      //     MP_RETURN_IF_ERROR(graph.AddPacketToInputStream(...));
202      //   }
203      //   for (const std::string& stream : streams) {
204      //     MP_RETURN_IF_ERROR(graph.CloseInputStream(stream));
205      //   }
206      //   MP_RETURN_IF_ERROR(graph.WaitUntilDone());
207      absl::Status StartRun(
208          const std::map<std::string, Packet>& extra_side_packets) {
209        return StartRun(extra_side_packets, {});
210      }
211    
```

<br/>

## 入力データの挿入

<br/>

`stream_name`<swm-token data-swm-token=":mediapipe/framework/calculator_graph.h:254:14:14:`  absl::Status AddPacketToInputStream(const std::string&amp; stream_name,`"/> で指定した入力ストリームにパケットを追加
<!-- NOTE-swimm-snippet: the lines below link your snippet to Swimm -->
### 📄 mediapipe/framework/calculator_graph.h
```c
245      // Add a Packet to a graph input stream based on the graph input stream add
246      // mode. If the mode is ADD_IF_NOT_FULL, the packet will not be added if any
247      // queue exceeds max_queue_size specified by the graph config and will return
248      // StatusUnavailable. The WAIT_TILL_NOT_FULL mode (default) will block until
249      // the queues fall below the max_queue_size before adding the packet. If the
250      // mode is max_queue_size is -1, then the packet is added regardless of the
251      // sizes of the queues in the graph. The input stream must have been specified
252      // in the configuration as a graph level input_stream. On error, nothing is
253      // added.
254      absl::Status AddPacketToInputStream(const std::string& stream_name,
255                                          const Packet& packet);
256    
```

<br/>

## 出力データに対するコールバック

<br/>

`stream_name`<swm-token data-swm-token=":mediapipe/framework/calculator_graph.h:159:8:8:`      const std::string&amp; stream_name,`"/>で指定した出力ストリームから出てきたパケットを引数に実行するコールバックの登録
<!-- NOTE-swimm-snippet: the lines below link your snippet to Swimm -->
### 📄 mediapipe/framework/calculator_graph.h
```c
152      // Observes the named output stream. packet_callback will be invoked on every
153      // packet emitted by the output stream. Can only be called before Run() or
154      // StartRun(). It is possible for packet_callback to be called until the
155      // object is destroyed, even if e.g. Cancel() or WaitUntilDone() have already
156      // been called. After this object is destroyed so is packet_callback.
157      // TODO: Rename to AddOutputStreamCallback.
158      absl::Status ObserveOutputStream(
159          const std::string& stream_name,
160          std::function<absl::Status(const Packet&)> packet_callback,
161          bool observe_timestamp_bounds = false);
162    
```

<br/>

# CalculatorGraphのデータ構造

CalculatorGraphの中で用いられる主なクラスに関して説明します。

[https://miro.com/app/board/uXjVMZlsp8U=/?moveToWidget=3458764549819273119&cot=14](https://miro.com/app/board/uXjVMZlsp8U=/?moveToWidget=3458764549819273119&cot=14)

## `ValidatedGraphConfig`<swm-token data-swm-token=":mediapipe/framework/validated_graph_config.h:192:2:2:`class ValidatedGraphConfig {`"/>

`CalculatorGraphConfig`<swm-token data-swm-token=":mediapipe/framework/calculator.proto:224:2:2:`message CalculatorGraphConfig {`"/>が正しいかどうか検査する。その中で GetContract を呼び出すことでCalulatorの入出力が正当かどうかのチェックも行う。

## `CalculatorNode`<swm-token data-swm-token=":mediapipe/framework/calculator_node.h:63:2:2:`class CalculatorNode {`"/>

グラフ内のノードに対応する。主なフィールドに次のようなものがある。

*   `CalculatorBase`<swm-token data-swm-token=":mediapipe/framework/calculator_base.h:75:2:2:`class CalculatorBase {`"/> Calculatorの実装

*   `InputStreamHandler`<swm-token data-swm-token=":mediapipe/framework/input_stream_handler.h:68:2:2:`class InputStreamHandler {`"/> ノードの入力ストリームの状態から実行すべきかどうかを判断する

*   `OutputStreamHandler`<swm-token data-swm-token=":mediapipe/framework/output_stream_handler.h:43:2:2:`class OutputStreamHandler {`"/> ノードの出力ストリームを次のノードの入力ストリームに伝播させる

*   `CalculatorContextManager`<swm-token data-swm-token=":mediapipe/framework/calculator_context_manager.h:36:2:2:`class CalculatorContextManager {`"/> Calculatorの実行時に参照される `CalculatorContext`<swm-token data-swm-token=":mediapipe/framework/calculator_context.h:44:2:2:`class CalculatorContext {`"/> を管理する

*   \*(ポインタ) `SchedulerQueue`<swm-token data-swm-token=":mediapipe/framework/scheduler_queue.h:38:2:2:`class SchedulerQueue : public TaskQueue {`"/> このノードが実行されるキューへの参照

## `Scheduler`<swm-token data-swm-token=":mediapipe/framework/scheduler.h:43:2:2:`class Scheduler {`"/>

ノードとノードが実行されるキューを管理する。`SchedulerQueue`<swm-token data-swm-token=":mediapipe/framework/scheduler_queue.h:38:2:2:`class SchedulerQueue : public TaskQueue {`"/> の配列（std::vector）をフィールドに持つ。

### `SchedulerQueue`<swm-token data-swm-token=":mediapipe/framework/scheduler_queue.h:38:2:2:`class SchedulerQueue : public TaskQueue {`"/>

ノードとコンテキストの組を要素とする優先度付きキューと `Executor`<swm-token data-swm-token=":mediapipe/framework/executor.h:43:2:2:`class Executor {`"/> を持つ。Executor上で優先度に従ってノードを実行する。

## `InputStreamManager`<swm-token data-swm-token=":mediapipe/framework/input_stream_manager.h:46:2:2:`class InputStreamManager {`"/>

ノードの入力ストリームに対応する。入力ストリームに流れるデータパケットをキューに保持し、InputStreamHandler から参照される。

## `OutputStreamManager`<swm-token data-swm-token=":mediapipe/framework/output_stream_manager.h:38:2:2:`class OutputStreamManager {`"/>

ノードの出力ストリームに対応する。次のノードのInputStreamHandlerへの参照とそのノードの中のどの入力ストリームと繋がっているかをMirrorという形で保持する。

# Packetの入力・実行・出力まで

## グラフレベルの入力からタスクのスケジューリングまで

<br/>

グラフレベルので入力
<!-- NOTE-swimm-snippet: the lines below link your snippet to Swimm -->
### 📄 mediapipe/framework/calculator_graph.cc
```c++
863    absl::Status CalculatorGraph::AddPacketToInputStream(
864        const std::string& stream_name, const Packet& packet) {
865      return AddPacketToInputStreamInternal(stream_name, packet);
866    }
```

<br/>

実装（Packetがconst&な場合と右辺値&&の場合があるのでtemplateを使って抽象化）

入力ストリームから対応するGraphInputStreamを参照してAddPacketを呼び出す。
<!-- NOTE-swimm-snippet: the lines below link your snippet to Swimm -->
### 📄 mediapipe/framework/calculator_graph.cc
```c++
886    // We avoid having two copies of this code for AddPacketToInputStream(
887    // const Packet&) and AddPacketToInputStream(Packet &&) by having this
888    // internal-only templated version.  T&& is a forwarding reference here, so
889    // std::forward will deduce the correct type as we pass along packet.
890    template <typename T>
891    absl::Status CalculatorGraph::AddPacketToInputStreamInternal(
892        const std::string& stream_name, T&& packet) {
893      std::unique_ptr<GraphInputStream>* stream =
894          mediapipe::FindOrNull(graph_input_streams_, stream_name);
895      RET_CHECK(stream).SetNoLogging() << absl::Substitute(
896          "AddPacketToInputStream called on input stream \"$0\" which is not a "
897          "graph input stream.",
898          stream_name);
899      int node_id = mediapipe::FindOrDie(graph_input_stream_node_ids_, stream_name);
900      CHECK_GE(node_id, validated_graph_->CalculatorInfos().size());
901      {
902        absl::MutexLock lock(&full_input_streams_mutex_);
903        if (full_input_streams_.empty()) {
904          return mediapipe::FailedPreconditionErrorBuilder(MEDIAPIPE_LOC)
905                 << "CalculatorGraph::AddPacketToInputStream() is called before "
906                    "StartRun()";
907        }
908        if (graph_input_stream_add_mode_ ==
909            GraphInputStreamAddMode::ADD_IF_NOT_FULL) {
910          if (has_error_) {
911            absl::Status error_status;
912            GetCombinedErrors("Graph has errors: ", &error_status);
913            return error_status;
914          }
915          // Return with StatusUnavailable if this stream is being throttled.
916          if (!full_input_streams_[node_id].empty()) {
917            return mediapipe::UnavailableErrorBuilder(MEDIAPIPE_LOC)
918                   << "Graph is throttled.";
919          }
920        } else if (graph_input_stream_add_mode_ ==
921                   GraphInputStreamAddMode::WAIT_TILL_NOT_FULL) {
922          // Wait until this stream is not being throttled.
923          // TODO: instead of checking has_error_, we could just check
924          // if the graph is done. That could also be indicated by returning an
925          // error from WaitUntilGraphInputStreamUnthrottled.
926          while (!has_error_ && !full_input_streams_[node_id].empty()) {
927            // TODO: allow waiting for a specific stream?
928            scheduler_.WaitUntilGraphInputStreamUnthrottled(
929                &full_input_streams_mutex_);
930          }
931          if (has_error_) {
932            absl::Status error_status;
933            GetCombinedErrors("Graph has errors: ", &error_status);
934            return error_status;
935          }
936        }
937      }
938    
939      // Adding profiling info for a new packet entering the graph.
940      const std::string* stream_id = &(*stream)->GetManager()->Name();
941      profiler_->LogEvent(TraceEvent(TraceEvent::PROCESS)
942                              .set_is_finish(true)
943                              .set_input_ts(packet.Timestamp())
944                              .set_stream_id(stream_id)
945                              .set_packet_ts(packet.Timestamp())
946                              .set_packet_data_id(&packet));
947    
948      // InputStreamManager is thread safe. GraphInputStream is not, so this method
949      // should not be called by multiple threads concurrently. Note that this could
950      // potentially lead to the max queue size being exceeded by one packet at most
951      // because we don't have the lock over the input stream.
952      (*stream)->AddPacket(std::forward<T>(packet));
953      if (has_error_) {
954        absl::Status error_status;
955        GetCombinedErrors("Graph has errors: ", &error_status);
956        return error_status;
957      }
958      (*stream)->PropagateUpdatesToMirrors();
959    
960      VLOG(2) << "Packet added directly to: " << stream_name;
961      // Note: one reason why we need to call the scheduler here is that we have
962      // re-throttled the graph input streams, and we may need to unthrottle them
963      // again if the graph is still idle. Unthrottling basically only lets in one
964      // packet at a time. TODO: add test.
965      scheduler_.AddedPacketToGraphInputStream();
966      return absl::OkStatus();
967    }
968    
```

<br/>

`OutputStreamShard`<swm-token data-swm-token=":mediapipe/framework/output_stream_shard.h:55:2:2:`class OutputStreamShard : public OutputStream {`"/> shard\_ （一時的なキュー）への追加
<!-- NOTE-swimm-snippet: the lines below link your snippet to Swimm -->
### 📄 mediapipe/framework/calculator_graph.h
```c
481        void AddPacket(const Packet& packet) { shard_.AddPacket(packet); }
482    
```

<br/>

OutputStreamManager::PropagateUpdatesToMirros()の呼び出し（次の入力ストリームへの伝播）
<!-- NOTE-swimm-snippet: the lines below link your snippet to Swimm -->
### 📄 mediapipe/framework/calculator_graph.cc
```c++
104    void CalculatorGraph::GraphInputStream::PropagateUpdatesToMirrors() {
105      manager_->PropagateUpdatesToMirrors(shard_.NextTimestampBound(), &shard_);
106    }
107    
```

<br/>

次の入力ストリームへの伝播

InputStreamHandlerを経由する。一つの出力ストリームに対してMirrorは複数存在する（複数ノードの入力になる可能性がある）。
<!-- NOTE-swimm-snippet: the lines below link your snippet to Swimm -->
### 📄 mediapipe/framework/output_stream_manager.cc
```c++
163    // TODO Consider moving the propagation logic to OutputStreamHandler.
164    void OutputStreamManager::PropagateUpdatesToMirrors(
165        Timestamp next_timestamp_bound, OutputStreamShard* output_stream_shard) {
166      CHECK(output_stream_shard);
167      {
168        if (next_timestamp_bound != Timestamp::Unset()) {
169          absl::MutexLock lock(&stream_mutex_);
170          next_timestamp_bound_ = next_timestamp_bound;
171          VLOG(3) << "Next timestamp bound for output " << output_stream_spec_.name
172                  << " is " << next_timestamp_bound_;
173        }
174      }
175      std::list<Packet>* packets_to_propagate = output_stream_shard->OutputQueue();
176      VLOG(3) << "Output stream: " << Name()
177              << " queue size: " << packets_to_propagate->size();
178      VLOG(3) << "Output stream: " << Name()
179              << " next timestamp: " << next_timestamp_bound;
180      bool add_packets = !packets_to_propagate->empty();
181      bool set_bound =
182          (next_timestamp_bound != Timestamp::Unset()) &&
183          (!add_packets ||
184           packets_to_propagate->back().Timestamp().NextAllowedInStream() !=
185               next_timestamp_bound);
186      int mirror_count = mirrors_.size();
187      for (int idx = 0; idx < mirror_count; ++idx) {
188        const Mirror& mirror = mirrors_[idx];
189        if (add_packets) {
190          // If the stream is the last element in mirrors_, moves packets from
191          // output_queue_. Otherwise, copies the packets.
192          if (idx == mirror_count - 1) {
193            mirror.input_stream_handler->MovePackets(mirror.id,
194                                                     packets_to_propagate);
195          } else {
196            mirror.input_stream_handler->AddPackets(mirror.id,
197                                                    *packets_to_propagate);
198          }
199        }
200        if (set_bound) {
201          mirror.input_stream_handler->SetNextTimestampBound(mirror.id,
202                                                             next_timestamp_bound);
203        }
204      }
205      // Clear out the packets.
206      packets_to_propagate->clear();
207    }
208    
```

<br/>

次の入力ストリーム（InputStreamManager）への追加

変数notifyに応じてコールバックが実行される。このコールバックを通じてノードのスケジューリングが行われる。
<!-- NOTE-swimm-snippet: the lines below link your snippet to Swimm -->
### 📄 mediapipe/framework/input_stream_handler.cc
```c++
260    void InputStreamHandler::AddPackets(CollectionItemId id,
261                                        const std::list<Packet>& packets) {
262      LogQueuedPackets(GetCalculatorContext(calculator_context_manager_),
263                       input_stream_managers_.Get(id), packets.back());
264      bool notify = false;
265      absl::Status result =
266          input_stream_managers_.Get(id)->AddPackets(packets, &notify);
267      if (!result.ok()) {
268        error_callback_(result);
269      }
270      if (notify) {
271        notification_();
272      }
273    }
274    
```

<br/>

内部メソッドの呼び出し
<!-- NOTE-swimm-snippet: the lines below link your snippet to Swimm -->
### 📄 mediapipe/framework/input_stream_manager.cc
```c++
83     absl::Status InputStreamManager::AddPackets(const std::list<Packet>& container,
84                                                 bool* notify) {
85       return AddOrMovePacketsInternal<const std::list<Packet>&>(container, notify);
86     }
87     
```

<br/>

入力ストリームキューへのパケットの追加

notifyがtrueになるのはキューが空から非空になったとき
<!-- NOTE-swimm-snippet: the lines below link your snippet to Swimm -->
### 📄 mediapipe/framework/input_stream_manager.cc
```c++
93     template <typename Container>
94     absl::Status InputStreamManager::AddOrMovePacketsInternal(Container container,
95                                                               bool* notify) {
96       *notify = false;
97       bool queue_became_non_empty = false;
98       bool queue_became_full = false;
99       {
100        // Scope to prevent locking the stream when notification is called.
101        absl::MutexLock stream_lock(&stream_mutex_);
102        if (closed_) {
103          return absl::OkStatus();
104        }
105        // Check if the queue was full before packets came in.
106        bool was_queue_full =
107            (max_queue_size_ != -1 && queue_.size() >= max_queue_size_);
108        // Check if the queue becomes non-empty.
109        queue_became_non_empty = queue_.empty() && !container.empty();
110        for (auto& packet : container) {
111          absl::Status result = packet_type_->Validate(packet);
112          if (!result.ok()) {
113            return tool::AddStatusPrefix(
114                absl::StrCat(
115                    "Packet type mismatch on a calculator receiving from stream \"",
116                    name_, "\": "),
117                result);
118          }
119    
120          const Timestamp timestamp = packet.Timestamp();
121          if (!timestamp.IsAllowedInStream()) {
122            return mediapipe::InvalidArgumentErrorBuilder(MEDIAPIPE_LOC)
123                   << "In stream \"" << name_
124                   << "\", timestamp not specified or set to illegal value: "
125                   << timestamp.DebugString();
126          }
127          if (enable_timestamps_) {
128            // Check that PostStream(), if used, is the only timestamp used.  This
129            // is also true for PreStream() but doesn't need to be checked because
130            // Timestamp::PreStream().NextAllowedInStream() is
131            // Timestamp::OneOverPostStream().
132            if (timestamp == Timestamp::PostStream() && num_packets_added_ > 0) {
133              return mediapipe::InvalidArgumentErrorBuilder(MEDIAPIPE_LOC)
134                     << "In stream \"" << name_
135                     << "\", a packet at Timestamp::PostStream() must be the only "
136                        "Packet in an InputStream.";
137            }
138            if (timestamp < next_timestamp_bound_) {
139              return mediapipe::InvalidArgumentErrorBuilder(MEDIAPIPE_LOC)
140                     << "Packet timestamp mismatch on a calculator receiving from "
141                        "stream \""
142                     << name_ << "\". Current minimum expected timestamp is "
143                     << next_timestamp_bound_.DebugString() << " but received "
144                     << timestamp.DebugString()
145                     << ". Are you using a custom InputStreamHandler? Note that "
146                        "some InputStreamHandlers allow timestamps that are not "
147                        "strictly monotonically increasing. See for example the "
148                        "ImmediateInputStreamHandler class comment.";
149            }
150          }
151          next_timestamp_bound_ = timestamp.NextAllowedInStream();
152    
153          // If the caller is MovePackets(), packet's underlying holder should be
154          // transferred into queue_. Otherwise, queue_ keeps a copy of the packet.
155          ++num_packets_added_;
156          VLOG(3) << "Input stream:" << name_
157                  << " has added packet at time: " << packet.Timestamp();
158          if (std::is_const<
159                  typename std::remove_reference<Container>::type>::value) {
160            queue_.emplace_back(packet);
161          } else {
162            queue_.emplace_back(std::move(packet));
163          }
164        }
165        queue_became_full = (!was_queue_full && max_queue_size_ != -1 &&
166                             queue_.size() >= max_queue_size_);
167        if (queue_.size() > 1) {
168          VLOG(3) << "Queue size greater than 1: stream name: " << name_
169                  << " queue_size: " << queue_.size();
170        }
171        VLOG(3) << "Input stream:" << name_
172                << " becomes non-empty status:" << queue_became_non_empty
173                << " Size: " << queue_.size();
174      }
175      if (queue_became_full) {
176        VLOG(3) << "Queue became full: " << Name();
177        becomes_full_callback_(this, &last_reported_stream_full_);
178      }
179      *notify = queue_became_non_empty;
180      return absl::OkStatus();
181    }
182    
```

<br/>

ここからnotification\_の定義を逆に追います。

<br/>

notification\_への代入
<!-- NOTE-swimm-snippet: the lines below link your snippet to Swimm -->
### 📄 mediapipe/framework/input_stream_handler.cc
```c++
69     void InputStreamHandler::PrepareForRun(
70         std::function<void()> headers_ready_callback,
71         std::function<void()> notification_callback,
72         std::function<void(CalculatorContext*)> schedule_callback,
73         std::function<void(absl::Status)> error_callback) {
74       headers_ready_callback_ = std::move(headers_ready_callback);
75       notification_ = std::move(notification_callback);
76       schedule_callback_ = std::move(schedule_callback);
77       error_callback_ = std::move(error_callback);
78       int unset_header_count = 0;
79       for (const auto& stream : input_stream_managers_) {
80         if (!stream->BackEdge()) {
81           ++unset_header_count;
82         }
83         stream->PrepareForRun();
84       }
85       unset_header_count_.store(unset_header_count, std::memory_order_relaxed);
86       prepared_context_for_close_ = false;
87     }
88     
```

<br/>

InputStreamHandler::PrepareForRun の呼び出し
<!-- NOTE-swimm-snippet: the lines below link your snippet to Swimm -->
### 📄 mediapipe/framework/calculator_node.cc
```c++
373    absl::Status CalculatorNode::PrepareForRun(
374        const std::map<std::string, Packet>& all_side_packets,
375        const std::map<std::string, Packet>& service_packets,
376        std::function<void()> ready_for_open_callback,
377        std::function<void()> source_node_opened_callback,
378        std::function<void(CalculatorContext*)> schedule_callback,
379        std::function<void(absl::Status)> error_callback,
380        CounterFactory* counter_factory) {
381      RET_CHECK(ready_for_open_callback) << "ready_for_open_callback is NULL";
382      RET_CHECK(schedule_callback) << "schedule_callback is NULL";
383      RET_CHECK(error_callback) << "error_callback is NULL";
384      calculator_state_->ResetBetweenRuns();
385    
386      ready_for_open_callback_ = std::move(ready_for_open_callback);
387      source_node_opened_callback_ = std::move(source_node_opened_callback);
388      input_stream_handler_->PrepareForRun(
389          [this]() { CalculatorNode::InputStreamHeadersReady(); },
390          [this]() { CalculatorNode::CheckIfBecameReady(); },
391          std::move(schedule_callback), error_callback);
392      output_stream_handler_->PrepareForRun(error_callback);
393    
394      const auto& contract = Contract();
395      input_side_packet_types_ = RemoveOmittedPacketTypes(
396          contract.InputSidePackets(), all_side_packets, validated_graph_);
397      MP_RETURN_IF_ERROR(input_side_packet_handler_.PrepareForRun(
398          input_side_packet_types_.get(), all_side_packets,
399          [this]() { CalculatorNode::InputSidePacketsReady(); },
400          std::move(error_callback)));
401      calculator_state_->SetInputSidePackets(
402          &input_side_packet_handler_.InputSidePackets());
403      calculator_state_->SetOutputSidePackets(output_side_packets_.get());
404      calculator_state_->SetCounterFactory(counter_factory);
405    
406      for (const auto& svc_req : contract.ServiceRequests()) {
407        const auto& req = svc_req.second;
408        auto it = service_packets.find(req.Service().key);
409        if (it == service_packets.end()) {
410          RET_CHECK(req.IsOptional())
411              << "required service '" << req.Service().key << "' was not provided";
412        } else {
413          MP_RETURN_IF_ERROR(
414              calculator_state_->SetServicePacket(req.Service(), it->second));
415        }
416      }
417    
418      MP_RETURN_IF_ERROR(calculator_context_manager_.PrepareForRun(std::bind(
419          &CalculatorNode::ConnectShardsToStreams, this, std::placeholders::_1)));
420    
421      ASSIGN_OR_RETURN(
422          auto calculator_factory,
423          CalculatorBaseRegistry::CreateByNameInNamespace(
424              validated_graph_->Package(), calculator_state_->CalculatorType()));
425      calculator_ = calculator_factory->CreateCalculator(
426          calculator_context_manager_.GetDefaultCalculatorContext());
427    
428      needs_to_close_ = false;
429    
430      {
431        absl::MutexLock status_lock(&status_mutex_);
432        status_ = kStatePrepared;
433        scheduling_state_ = kIdle;
434        current_in_flight_ = 0;
435        input_stream_headers_ready_called_ = false;
436        input_side_packets_ready_called_ = false;
437        input_stream_headers_ready_ =
438            (input_stream_handler_->UnsetHeaderCount() == 0);
439        input_side_packets_ready_ =
440            (input_side_packet_handler_.MissingInputSidePacketCount() == 0);
441      }
442      return absl::OkStatus();
443    }
```

<br/>

notifaction\_として呼ばれるメソッド
<!-- NOTE-swimm-snippet: the lines below link your snippet to Swimm -->
### 📄 mediapipe/framework/calculator_node.cc
```c++
723    void CalculatorNode::CheckIfBecameReady() {
724      {
725        absl::MutexLock lock(&status_mutex_);
726        // Doesn't check if status_ is kStateActive since the function can only be
727        // invoked by non-source nodes.
728        if (status_ != kStateOpened) {
729          return;
730        }
731        if (scheduling_state_ == kIdle && current_in_flight_ < max_in_flight_) {
732          scheduling_state_ = kScheduling;
733        } else {
734          if (scheduling_state_ == kScheduling) {
735            // Changes the state to scheduling pending if another thread is doing
736            // the scheduling.
737            scheduling_state_ = kSchedulingPending;
738          }
739          return;
740        }
741      }
742      SchedulingLoop();
743    }
744    
```

<br/>

続き

特定の条件に達するまでScheduleInvocationsを呼び続ける
<!-- NOTE-swimm-snippet: the lines below link your snippet to Swimm -->
### 📄 mediapipe/framework/calculator_node.cc
```c++
653    void CalculatorNode::SchedulingLoop() {
654      int max_allowance = 0;
655      {
656        absl::MutexLock lock(&status_mutex_);
657        if (status_ == kStateClosed) {
658          scheduling_state_ = kIdle;
659          return;
660        }
661        max_allowance = max_in_flight_ - current_in_flight_;
662      }
663      while (true) {
664        Timestamp input_bound;
665        // input_bound is set to a meaningful value iff the latest readiness of the
666        // node is kNotReady when ScheduleInvocations() returns.
667        input_stream_handler_->ScheduleInvocations(max_allowance, &input_bound);
668        if (input_bound != Timestamp::Unset()) {
669          // Updates the minimum timestamp for which a new packet could possibly
670          // arrive.
671          output_stream_handler_->UpdateTaskTimestampBound(input_bound);
672        }
673    
674        {
675          absl::MutexLock lock(&status_mutex_);
676          if (scheduling_state_ == kSchedulingPending &&
677              current_in_flight_ < max_in_flight_) {
678            max_allowance = max_in_flight_ - current_in_flight_;
679            scheduling_state_ = kScheduling;
680          } else {
681            scheduling_state_ = kIdle;
682            break;
683          }
684        }
685      }
686    }
687    
```

<br/>

実行準備が完了しているかどうか `GetNodeReadiness`<swm-token data-swm-token=":mediapipe/framework/stream_handler/default_input_stream_handler.cc:59:4:4:`NodeReadiness DefaultInputStreamHandler::GetNodeReadiness(`"/> の結果に基づいてCaluclatorContextに入力を設定 `FillInputSet`<swm-token data-swm-token=":mediapipe/framework/input_stream_handler.h:219:3:3:`    void FillInputSet(Timestamp input_timestamp,`"/>して 条件を満たせば schedule\_callback\_ を呼び出す

GetNodeReadiness と FillInputSet は virtual methodになっていてノードごとに指定するInputStreamHandlerの種類によって挙動が変わる。
<!-- NOTE-swimm-snippet: the lines below link your snippet to Swimm -->
### 📄 mediapipe/framework/input_stream_handler.cc
```c++
147    bool InputStreamHandler::ScheduleInvocations(int max_allowance,
148                                                 Timestamp* input_bound) {
149      *input_bound = Timestamp::Unset();
150      Timestamp min_stream_timestamp = Timestamp::Done();
151      if (input_stream_managers_.NumEntries() == 0) {
152        // A source node doesn't require any input packets.
153        CalculatorContext* default_context =
154            calculator_context_manager_->GetDefaultCalculatorContext();
155        schedule_callback_(default_context);
156        return true;
157      }
158      int invocations_scheduled = 0;
159      while (invocations_scheduled < max_allowance) {
160        NodeReadiness node_readiness = GetNodeReadiness(&min_stream_timestamp);
161        // Sets *input_bound iff the latest node readiness is kNotReady before the
162        // function returns regardless of how many invocations have been scheduled.
163        if (node_readiness == NodeReadiness::kNotReady) {
164          if (batch_size_ > 1 &&
165              calculator_context_manager_->ContextHasInputTimestamp(
166                  *calculator_context_manager_->GetDefaultCalculatorContext())) {
167            // When batching is in progress, input_bound stays equal to the first
168            // timestamp in the calculator context. This allows timestamp
169            // propagation to be performed only for the first timestamp, and
170            // prevents propagation for the subsequent inputs.
171            *input_bound =
172                calculator_context_manager_->GetDefaultCalculatorContext()
173                    ->InputTimestamp();
174          } else {
175            *input_bound = min_stream_timestamp;
176          }
177          CalculatorContext* default_context =
178              calculator_context_manager_->GetDefaultCalculatorContext();
179          mediapipe::LogEvent(default_context->GetProfilingContext(),
180                              TraceEvent(TraceEvent::NOT_READY)
181                                  .set_node_id(default_context->NodeId()));
182          break;
183        } else if (node_readiness == NodeReadiness::kReadyForProcess) {
184          CalculatorContext* calculator_context =
185              calculator_context_manager_->PrepareCalculatorContext(
186                  min_stream_timestamp);
187          calculator_context_manager_->PushInputTimestampToContext(
188              calculator_context, min_stream_timestamp);
189          if (!late_preparation_) {
190            FillInputSet(min_stream_timestamp, &calculator_context->Inputs());
191          }
192          if (calculator_context_manager_->NumberOfContextTimestamps(
193                  *calculator_context) == batch_size_) {
194            schedule_callback_(calculator_context);
195            ++invocations_scheduled;
196          }
197          mediapipe::LogEvent(calculator_context->GetProfilingContext(),
198                              TraceEvent(TraceEvent::READY_FOR_PROCESS)
199                                  .set_node_id(calculator_context->NodeId()));
200        } else {
201          CHECK(node_readiness == NodeReadiness::kReadyForClose);
202          // If any parallel invocations are in progress or a calculator context has
203          // been prepared for Close(), we shouldn't prepare another calculator
204          // context for Close().
205          if (calculator_context_manager_->HasActiveContexts() ||
206              prepared_context_for_close_) {
207            break;
208          }
209          // If there is an incomplete batch of input sets in the calculator
210          // context, it gets scheduled when the calculator is ready for close.
211          CalculatorContext* default_context =
212              calculator_context_manager_->GetDefaultCalculatorContext();
213          calculator_context_manager_->PushInputTimestampToContext(
214              default_context, Timestamp::Done());
215          schedule_callback_(default_context);
216          ++invocations_scheduled;
217          prepared_context_for_close_ = true;
218          mediapipe::LogEvent(default_context->GetProfilingContext(),
219                              TraceEvent(TraceEvent::READY_FOR_CLOSE)
220                                  .set_node_id(default_context->NodeId()));
221          break;
222        }
223      }
224      return invocations_scheduled > 0;
225    }
226    
```

<br/>

schedule\_callback\_の定義箇所
<!-- NOTE-swimm-snippet: the lines below link your snippet to Swimm -->
### 📄 mediapipe/framework/calculator_graph.cc
```c++
753      for (auto& node : nodes_) {
754        InputStreamManager::QueueSizeCallback queue_size_callback =
755            std::bind(&CalculatorGraph::UpdateThrottledNodes, this,
756                      std::placeholders::_1, std::placeholders::_2);
757        node->SetQueueSizeCallbacks(queue_size_callback, queue_size_callback);
758        scheduler_.AssignNodeToSchedulerQueue(node.get());
759        // TODO: update calculator node to use GraphServiceManager
760        // instead of service packets?
761        const absl::Status result = node->PrepareForRun(
762            current_run_side_packets_, service_manager_.ServicePackets(),
763            std::bind(&internal::Scheduler::ScheduleNodeForOpen, &scheduler_,
764                      node.get()),
765            std::bind(&internal::Scheduler::AddNodeToSourcesQueue, &scheduler_,
766                      node.get()),
767            std::bind(&internal::Scheduler::ScheduleNodeIfNotThrottled, &scheduler_,
768                      node.get(), std::placeholders::_1),
769            std::bind(&CalculatorGraph::RecordError, this, std::placeholders::_1),
770            counter_factory_.get());
771        if (!result.ok()) {
772          // Collect as many errors as we can before failing.
773          RecordError(result);
774        }
775      }
```

<br/>

schedule\_callback\_ としてこれが呼ばれる

ノードに紐づくSchedulerQueueに対してAddNodeを呼び出す
<!-- NOTE-swimm-snippet: the lines below link your snippet to Swimm -->
### 📄 mediapipe/framework/scheduler.cc
```c++
334    void Scheduler::ScheduleNodeIfNotThrottled(
335        CalculatorNode* node, CalculatorContext* calculator_context) {
336      DCHECK(node);
337      DCHECK(calculator_context);
338      if (!graph_->IsNodeThrottled(node->Id())) {
339        node->GetSchedulerQueue()->AddNode(node, calculator_context);
340      }
341    }
342    
```

<br/>

キューへノードとコンテキストを追加
<!-- NOTE-swimm-snippet: the lines below link your snippet to Swimm -->
### 📄 mediapipe/framework/scheduler_queue.cc
```c++
110    void SchedulerQueue::AddNode(CalculatorNode* node, CalculatorContext* cc) {
111      // TODO: If the node isn't successfully scheduled, we must properly
112      // handle the pending calculator context.
113      if (shared_->has_error) {
114        return;
115      }
116      if (!node->TryToBeginScheduling()) {
117        // Only happens when the framework tries to schedule an unthrottled source
118        // node while it's running. For non-source nodes, if a calculator context is
119        // prepared, it is committed to be scheduled.
120        CHECK(node->IsSource()) << node->DebugName();
121        return;
122      }
123      AddItemToQueue(Item(node, cc));
124    }
125    
```

<br/>

キューへのpushとExecutorへ投げるべきタスク数（だいたいキュー内の要素数）分、Executor::AddTask を呼び出す
<!-- NOTE-swimm-snippet: the lines below link your snippet to Swimm -->
### 📄 mediapipe/framework/scheduler_queue.cc
```c++
133    void SchedulerQueue::AddItemToQueue(Item&& item) {
134      const CalculatorNode* node = item.Node();
135      bool was_idle;
136      int tasks_to_add = 0;
137      {
138        absl::MutexLock lock(&mutex_);
139        was_idle = IsIdle();
140        queue_.push(item);
141        ++num_tasks_to_add_;
142        VLOG(4) << node->DebugName() << " was added to the scheduler queue.";
143    
144        // Now grab the tasks to execute while still holding the lock. This will
145        // gather any waiting tasks, in addition to the one we just added.
146        if (running_count_ > 0) {
147          tasks_to_add = GetTasksToSubmitToExecutor();
148        }
149      }
150      if (was_idle && idle_callback_) {
151        // Became not idle.
152        idle_callback_(false);
153      }
154      // Note: this should be done after calling idle_callback_(false) above.
155      // This ensures that we never get an idle_callback_(true) that is not
156      // preceded by the corresponding idle_callback_(false). See the comments on
157      // SetIdleCallback for details.
158      while (tasks_to_add > 0) {
159        executor_->AddTask(this);
160        --tasks_to_add;
161      }
162    }
163    
```

<br/>

Executor上でTaskQueue （SchedulerQueueの基底クラス）のRunNextTask()を実行するようにスケジューリング
<!-- NOTE-swimm-snippet: the lines below link your snippet to Swimm -->
### 📄 mediapipe/framework/executor.h
```c
47       // A registered Executor subclass must implement the static factory method
48       // Create.  The Executor subclass cannot be registered without it.
49       //
50       // static absl::StatusOr<Executor*> Create(
51       //     const MediaPipeOptions& extendable_options);
52       //
53       // Create validates extendable_options, then calls the constructor, and
54       // returns the newly allocated Executor object.
55     
56       // The scheduler queue calls this method to tell the executor that it has
57       // a new task to run. The executor should use its execution mechanism to
58       // invoke task_queue->RunNextTask.
59       virtual void AddTask(TaskQueue* task_queue) {
60         Schedule([task_queue] { task_queue->RunNextTask(); });
61       }
62     
63       // Schedule the specified "task" for execution in this executor.
64       virtual void Schedule(std::function<void()> task) = 0;
65     };
```

<br/>

## ソースノードのスケジューリング

<br/>

CalculatorGraphの実行開始まで遡り、起動はScheduler::Startから行われる
<!-- NOTE-swimm-snippet: the lines below link your snippet to Swimm -->
### 📄 mediapipe/framework/calculator_graph.cc
```c++
541    absl::Status CalculatorGraph::StartRun(
542        const std::map<std::string, Packet>& extra_side_packets,
543        const std::map<std::string, Packet>& stream_headers) {
544      RET_CHECK(initialized_).SetNoLogging()
545          << "CalculatorGraph is not initialized.";
546      MP_RETURN_IF_ERROR(PrepareForRun(extra_side_packets, stream_headers));
547      MP_RETURN_IF_ERROR(profiler_->Start(executors_[""].get()));
548      scheduler_.Start();
549      return absl::OkStatus();
550    }
551    
```

<br/>

HandleIdle と SubmitWaitingTasksOnQueues が呼ばれる
<!-- NOTE-swimm-snippet: the lines below link your snippet to Swimm -->
### 📄 mediapipe/framework/scheduler.cc
```c++
196    void Scheduler::Start() {
197      VLOG(2) << "Starting scheduler";
198      shared_.timer.StartRun();
199      {
200        absl::MutexLock lock(&state_mutex_);
201        CHECK_EQ(state_, STATE_NOT_STARTED);
202        state_ = STATE_RUNNING;
203        SetQueuesRunning(true);
204    
205        // Get the ball rolling.
206        HandleIdle();
207      }
208      SubmitWaitingTasksOnQueues();
209    }
210    
```

<br/>

ソースノード（入力がないノード）で実行可能なものがあればスケジューリングする
<!-- NOTE-swimm-snippet: the lines below link your snippet to Swimm -->
### 📄 mediapipe/framework/scheduler.cc
```c++
117    // Note: state_mutex_ is held when this function is entered or
118    // exited.
119    void Scheduler::HandleIdle() {
120      if (++handling_idle_ > 1) {
121        // Someone is already inside this method.
122        // Note: This can happen in the sections below where we unlock the mutex
123        // and make more nodes runnable: the nodes can run and become idle again
124        // while this method is in progress. In that case, the resulting calls to
125        // HandleIdle are ignored, which is ok because the original method will
126        // run the loop again.
127        VLOG(2) << "HandleIdle: already in progress";
128        return;
129      }
130    
131      while (IsIdle() && (state_ == STATE_RUNNING || state_ == STATE_CANCELLING)) {
132        // Remove active sources that are closed.
133        CleanupActiveSources();
134    
135        // Quit if we have errors, or if there are no more packet sources.
136        if (shared_.has_error ||
137            (active_sources_.empty() && sources_queue_.empty() &&
138             graph_input_streams_closed_)) {
139          VLOG(2) << "HandleIdle: quitting";
140          Quit();
141          break;
142        }
143    
144        // See if we can schedule the next layer of source nodes.
145        if (active_sources_.empty() && !sources_queue_.empty()) {
146          VLOG(2) << "HandleIdle: activating sources";
147          // Note: TryToScheduleNextSourceLayer unlocks and locks state_mutex_
148          // internally.
149          bool did_activate = TryToScheduleNextSourceLayer();
150          CHECK(did_activate || active_sources_.empty());
151          continue;
152        }
153    
154        // See if we can unthrottle some source nodes or graph input streams to
155        // break deadlock. If we are still idle and there are active source nodes,
156        // they must be throttled.
157        if (!active_sources_.empty() || throttled_graph_input_stream_count_ > 0) {
158          VLOG(2) << "HandleIdle: unthrottling";
159          state_mutex_.Unlock();
160          bool did_unthrottle = graph_->UnthrottleSources();
161          state_mutex_.Lock();
162          if (did_unthrottle) {
163            continue;
164          }
165        }
166    
167        // If HandleIdle has been called again, then continue scheduling.
168        if (handling_idle_ > 1) {
169          handling_idle_ = 1;
170          continue;
171        }
172    
173        // Nothing left to do.
174        break;
175      }
176    
177      handling_idle_ = 0;
178    }
179    
```

<br/>

各SchedulerQueueで実行を待っているタスクをExecutorに投げる
<!-- NOTE-swimm-snippet: the lines below link your snippet to Swimm -->
### 📄 mediapipe/framework/scheduler.cc
```c++
111    void Scheduler::SubmitWaitingTasksOnQueues() {
112      for (auto queue : scheduler_queues_) {
113        queue->SubmitWaitingTasksToExecutor();
114      }
115    }
```

<br/>

キューに残っているタスクをExecutorに投げる
<!-- NOTE-swimm-snippet: the lines below link your snippet to Swimm -->
### 📄 mediapipe/framework/scheduler_queue.cc
```c++
171    void SchedulerQueue::SubmitWaitingTasksToExecutor() {
172      // If a node is added to the scheduler queue while the queue is not running,
173      // we do not immediately submit tasks to the executor. Here we check for any
174      // such waiting tasks, and submit them.
175      int tasks_to_add = 0;
176      {
177        absl::MutexLock lock(&mutex_);
178        if (running_count_ > 0) {
179          tasks_to_add = GetTasksToSubmitToExecutor();
180        }
181      }
182      while (tasks_to_add > 0) {
183        executor_->AddTask(this);
184        --tasks_to_add;
185      }
186    }
187    
```

<br/>

## スケジュールされたタスクの実行

デフォルトで初期化時にスレッドプールで実行用のスレッドが立ち上がります。その上で実行されるものに関して追っていきます。

<br/>

スケジュールされたタスクのエントリーポイント

キューから要素を取り除き、ノードの状態によってOpenCalculatorNode, もしくはRunCalculatorNodeを実行する。<br/>
（説明端折ってるが、Calculatorの初期化（Open）もCalculatorGraph::StartRunの最初にExecutor上で実行するようにスケジューリングされる）
<!-- NOTE-swimm-snippet: the lines below link your snippet to Swimm -->
### 📄 mediapipe/framework/scheduler_queue.cc
```c++
188    void SchedulerQueue::RunNextTask() {
189      CalculatorNode* node;
190      CalculatorContext* calculator_context;
191      bool is_open_node;
192      {
193        absl::MutexLock lock(&mutex_);
194    
195        CHECK(!queue_.empty()) << "Called RunNextTask when the queue is empty. "
196                                  "This should not happen.";
197    
198        node = queue_.top().Node();
199        calculator_context = queue_.top().Context();
200        is_open_node = queue_.top().IsOpenNode();
201        queue_.pop();
202    
203        CHECK(!node->Closed())
204            << "Scheduled a node that was closed. This should not happen.";
205      }
206    
207      // On iOS, calculators may rely on the existence of an autorelease pool
208      // (either directly, or because system code they call does). We do not
209      // want to rely on executors setting up an autorelease pool for us (e.g.
210      // an executor creating standard pthread will not, by default), so we
211      // do it here to ensure all executors are covered.
212      AUTORELEASEPOOL {
213        if (is_open_node) {
214          DCHECK(!calculator_context);
215          OpenCalculatorNode(node);
216        } else {
217          RunCalculatorNode(node, calculator_context);
218        }
219      }
220    
221      bool is_idle;
222      {
223        absl::MutexLock lock(&mutex_);
224        DCHECK_GT(num_pending_tasks_, 0);
225        --num_pending_tasks_;
226        is_idle = IsIdle();
227      }
228      if (is_idle && idle_callback_) {
229        // Became idle.
230        idle_callback_(true);
231      }
232    }
233    
```

<br/>

CalculatorNode::ProcessNodeを呼び出し、最後にCalculatorNode::EndSchedulingを実行する
<!-- NOTE-swimm-snippet: the lines below link your snippet to Swimm -->
### 📄 mediapipe/framework/scheduler_queue.cc
```c++
234    void SchedulerQueue::RunCalculatorNode(CalculatorNode* node,
235                                           CalculatorContext* cc) {
236      VLOG(3) << "Running " << node->DebugName();
237    
238      // If we are in the process of stopping the graph (due to tool::StatusStop()
239      // from a non-source node or due to CalculatorGraph::CloseAllPacketSources),
240      // we should not run any more sources.  Close the node if it is a source.
241      if (shared_->stopping && node->IsSource()) {
242        VLOG(4) << "Closing " << node->DebugName() << " due to StatusStop().";
243        int64 start_time = shared_->timer.StartNode();
244        // It's OK to not reset/release the prepared CalculatorContext since a
245        // source node always reuses the same CalculatorContext and Close() doesn't
246        // access any inputs.
247        // TODO: Should we pass tool::StatusStop() in this case?
248        const absl::Status result =
249            node->CloseNode(absl::OkStatus(), /*graph_run_ended=*/false);
250        shared_->timer.EndNode(start_time);
251        if (!result.ok()) {
252          VLOG(3) << node->DebugName()
253                  << " had an error while closing due to StatusStop()!";
254          shared_->error_callback(result);
255        }
256      } else {
257        // Note that we don't need a lock because only one thread can execute this
258        // due to the lock on running_nodes.
259        int64 start_time = shared_->timer.StartNode();
260        const absl::Status result = node->ProcessNode(cc);
261        shared_->timer.EndNode(start_time);
262    
263        if (!result.ok()) {
264          if (result == tool::StatusStop()) {
265            // Check if StatusStop was returned by a non-source node. This means
266            // that all sources will be closed and no further sources should be
267            // scheduled. The graph will be terminated as soon as its scheduler
268            // queue becomes empty.
269            CHECK(!node->IsSource());  // ProcessNode takes care of StatusStop()
270                                       // from sources.
271            shared_->stopping = true;
272          } else {
273            // If we have an error in this calculator.
274            VLOG(3) << node->DebugName() << " had an error!";
275            shared_->error_callback(result);
276          }
277        }
278      }
279    
280      VLOG(4) << "Done running " << node->DebugName();
281      node->EndScheduling();
282    }
283    
```

<br/>

ここでようやく実際の処理であるCalculatorBase::Process を呼び出す。その後、OutputStreamManager::PostProcessによって出力ストリームに流れたパケットを次のノードに伝播させる。
<!-- NOTE-swimm-snippet: the lines below link your snippet to Swimm -->
### 📄 mediapipe/framework/calculator_node.cc
```c++
798    absl::Status CalculatorNode::ProcessNode(
799        CalculatorContext* calculator_context) {
800      if (IsSource()) {
801        // This is a source Calculator.
802        if (Closed()) {
803          return absl::OkStatus();
804        }
805    
806        const Timestamp input_timestamp = calculator_context->InputTimestamp();
807    
808        OutputStreamShardSet* outputs = &calculator_context->Outputs();
809        output_stream_handler_->PrepareOutputs(input_timestamp, outputs);
810    
811        VLOG(2) << "Calling Calculator::Process() for node: " << DebugName();
812        absl::Status result;
813    
814        {
815          MEDIAPIPE_PROFILING(PROCESS, calculator_context);
816          LegacyCalculatorSupport::Scoped<CalculatorContext> s(calculator_context);
817          result = calculator_->Process(calculator_context);
818        }
819    
820        bool node_stopped = false;
821        if (!result.ok()) {
822          if (result == tool::StatusStop()) {
823            // Needs to call CloseNode().
824            node_stopped = true;
825          } else {
826            return mediapipe::StatusBuilder(result, MEDIAPIPE_LOC).SetPrepend()
827                   << absl::Substitute(
828                          "Calculator::Process() for node \"$0\" failed: ",
829                          DebugName());
830          }
831        }
832        output_stream_handler_->PostProcess(input_timestamp);
833        if (node_stopped) {
834          MP_RETURN_IF_ERROR(
835              CloseNode(absl::OkStatus(), /*graph_run_ended=*/false));
836        }
837        return absl::OkStatus();
838      } else {
839        // This is not a source Calculator.
840        InputStreamShardSet* const inputs = &calculator_context->Inputs();
841        OutputStreamShardSet* const outputs = &calculator_context->Outputs();
842        absl::Status result =
843            absl::InternalError("Calculator context has no input packets.");
844    
845        int num_invocations = calculator_context_manager_.NumberOfContextTimestamps(
846            *calculator_context);
847        RET_CHECK(num_invocations <= 1 || max_in_flight_ <= 1)
848            << "num_invocations:" << num_invocations
849            << ", max_in_flight_:" << max_in_flight_;
850        for (int i = 0; i < num_invocations; ++i) {
851          const Timestamp input_timestamp = calculator_context->InputTimestamp();
852          // The node is ready for Process().
853          if (input_timestamp.IsAllowedInStream()) {
854            input_stream_handler_->FinalizeInputSet(input_timestamp, inputs);
855            output_stream_handler_->PrepareOutputs(input_timestamp, outputs);
856    
857            VLOG(2) << "Calling Calculator::Process() for node: " << DebugName()
858                    << " timestamp: " << input_timestamp;
859    
860            if (OutputsAreConstant(calculator_context)) {
861              // Do nothing.
862              result = absl::OkStatus();
863            } else {
864              MEDIAPIPE_PROFILING(PROCESS, calculator_context);
865              LegacyCalculatorSupport::Scoped<CalculatorContext> s(
866                  calculator_context);
867              result = calculator_->Process(calculator_context);
868            }
869    
870            VLOG(2) << "Called Calculator::Process() for node: " << DebugName()
871                    << " timestamp: " << input_timestamp;
872    
873            // Removes one packet from each shard and progresses to the next input
874            // timestamp.
875            input_stream_handler_->ClearCurrentInputs(calculator_context);
876    
877            // Nodes are allowed to return StatusStop() to cause the termination
878            // of the graph. This is different from an error in that it will
879            // ensure that all sources will be closed and that packets in input
880            // streams will be processed before the graph is terminated.
881            if (!result.ok() && result != tool::StatusStop()) {
882              return mediapipe::StatusBuilder(result, MEDIAPIPE_LOC).SetPrepend()
883                     << absl::Substitute(
884                            "Calculator::Process() for node \"$0\" failed: ",
885                            DebugName());
886            }
887            output_stream_handler_->PostProcess(input_timestamp);
888            if (result == tool::StatusStop()) {
889              return result;
890            }
891          } else if (input_timestamp == Timestamp::Done()) {
892            // Some or all the input streams are closed and there are not enough
893            // open input streams for Process(). So this node needs to be closed
894            // too.
895            // If the streams are closed, there shouldn't be more input.
896            CHECK_EQ(calculator_context_manager_.NumberOfContextTimestamps(
897                         *calculator_context),
898                     1);
899            return CloseNode(absl::OkStatus(), /*graph_run_ended=*/false);
900          } else {
901            RET_CHECK_FAIL()
902                << "Invalid input timestamp in ProcessNode(). timestamp: "
903                << input_timestamp;
904          }
905        }
906        return result;
907      }
908    }
909    
```

<br/>

（一旦calculator\_run\_in\_parallel\_がtrueな場合は無視して）パケットを伝播
<!-- NOTE-swimm-snippet: the lines below link your snippet to Swimm -->
### 📄 mediapipe/framework/output_stream_handler.cc
```c++
95     void OutputStreamHandler::PostProcess(Timestamp input_timestamp) {
96       if (!calculator_run_in_parallel_) {
97         CalculatorContext* default_context =
98             calculator_context_manager_->GetDefaultCalculatorContext();
99         PropagateOutputPackets(input_timestamp, &default_context->Outputs());
100        return;
101      }
102      {
103        absl::MutexLock lock(&timestamp_mutex_);
104        completed_input_timestamps_.insert(input_timestamp);
105        if (propagation_state_ == kPropagatingBound) {
106          propagation_state_ = kPropagationPending;
107          return;
108        }
109        if (propagation_state_ != kIdle) {
110          return;
111        }
112        PropagationLoop();
113      }
114    }
115    
```

<br/>

各OutputStreamManagerからパケットを伝播

ここから先はグラフ入力ストリームのフローと同じ
<!-- NOTE-swimm-snippet: the lines below link your snippet to Swimm -->
### 📄 mediapipe/framework/output_stream_handler.cc
```c++
150    void OutputStreamHandler::PropagateOutputPackets(
151        Timestamp input_timestamp, OutputStreamShardSet* output_shards) {
152      CHECK(output_shards);
153      for (CollectionItemId id = output_stream_managers_.BeginId();
154           id < output_stream_managers_.EndId(); ++id) {
155        OutputStreamManager* manager = output_stream_managers_.Get(id);
156        if (manager->IsClosed()) {
157          continue;
158        }
159        OutputStreamShard* output = &output_shards->Get(id);
160        const Timestamp output_bound =
161            manager->ComputeOutputTimestampBound(*output, input_timestamp);
162        manager->PropagateUpdatesToMirrors(output_bound, output);
163        if (output->IsClosed()) {
164          manager->Close();
165        }
166      }
167    }
168    
```

<br/>

## 出力ストリームに対するコールバックが呼ばれるまで

<br/>

OutputStreamObserverとしてコールバックを登録
<!-- NOTE-swimm-snippet: the lines below link your snippet to Swimm -->
### 📄 mediapipe/framework/calculator_graph.cc
```c++
453    absl::Status CalculatorGraph::ObserveOutputStream(
454        const std::string& stream_name,
455        std::function<absl::Status(const Packet&)> packet_callback,
456        bool observe_timestamp_bounds) {
457      RET_CHECK(initialized_).SetNoLogging()
458          << "CalculatorGraph is not initialized.";
459      // TODO Allow output observers to be attached by graph level
460      // tag/index.
461      int output_stream_index = validated_graph_->OutputStreamIndex(stream_name);
462      if (output_stream_index < 0) {
463        return mediapipe::NotFoundErrorBuilder(MEDIAPIPE_LOC)
464               << "Unable to attach observer to output stream \"" << stream_name
465               << "\" because it doesn't exist.";
466      }
467      auto observer = absl::make_unique<internal::OutputStreamObserver>();
468      MP_RETURN_IF_ERROR(observer->Initialize(
469          stream_name, &any_packet_type_, std::move(packet_callback),
470          &output_stream_managers_[output_stream_index], observe_timestamp_bounds));
471      graph_output_streams_.push_back(std::move(observer));
472      return absl::OkStatus();
473    }
474    
```

<br/>

コールバックの初期化と基底クラス（GraphOutputStream）のInitializeメソッドの呼び出し
<!-- NOTE-swimm-snippet: the lines below link your snippet to Swimm -->
### 📄 mediapipe/framework/graph_output_stream.cc
```c++
56     absl::Status OutputStreamObserver::Initialize(
57         const std::string& stream_name, const PacketType* packet_type,
58         std::function<absl::Status(const Packet&)> packet_callback,
59         OutputStreamManager* output_stream_manager, bool observe_timestamp_bounds) {
60       RET_CHECK(output_stream_manager);
61     
62       packet_callback_ = std::move(packet_callback);
63       observe_timestamp_bounds_ = observe_timestamp_bounds;
64       return GraphOutputStream::Initialize(stream_name, packet_type,
65                                            output_stream_manager,
66                                            observe_timestamp_bounds);
67     }
68     
```

<br/>

OutputStreamManagerに次の入力としてOutputStreamObserverを登録

InputStreamHandler, InputStreamManagerを経由するのでそれぞれ初期化する。
<!-- NOTE-swimm-snippet: the lines below link your snippet to Swimm -->
### 📄 mediapipe/framework/graph_output_stream.cc
```c++
24     absl::Status GraphOutputStream::Initialize(
25         const std::string& stream_name, const PacketType* packet_type,
26         OutputStreamManager* output_stream_manager, bool observe_timestamp_bounds) {
27       RET_CHECK(output_stream_manager);
28     
29       // Initializes input_stream_handler_ with one input stream as the observer.
30       proto_ns::RepeatedPtrField<ProtoString> input_stream_field;
31       input_stream_field.Add()->assign(stream_name);
32       std::shared_ptr<tool::TagMap> tag_map =
33           tool::TagMap::Create(input_stream_field).value();
34       input_stream_handler_ = absl::make_unique<GraphOutputStreamHandler>(
35           tag_map, /*cc_manager=*/nullptr, MediaPipeOptions(),
36           /*calculator_run_in_parallel=*/false);
37       input_stream_handler_->SetProcessTimestampBounds(observe_timestamp_bounds);
38       const CollectionItemId& id = tag_map->BeginId();
39       input_stream_ = absl::make_unique<InputStreamManager>();
40       MP_RETURN_IF_ERROR(
41           input_stream_->Initialize(stream_name, packet_type, /*back_edge=*/false));
42       MP_RETURN_IF_ERROR(input_stream_handler_->InitializeInputStreamManagers(
43           input_stream_.get()));
44       output_stream_manager->AddMirror(input_stream_handler_.get(), id);
45       return absl::OkStatus();
46     }
47     
```

<br/>

このInputStreamHandlerはnotification\_ コールバックが通常のノードとは異なる。

<br/>

notification コールバックの登録
<!-- NOTE-swimm-snippet: the lines below link your snippet to Swimm -->
### 📄 mediapipe/framework/graph_output_stream.cc
```c++
48     void GraphOutputStream::PrepareForRun(
49         std::function<void()> notification_callback,
50         std::function<void(absl::Status)> error_callback) {
51       input_stream_handler_->PrepareForRun(
52           /*headers_ready_callback=*/[] {}, std::move(notification_callback),
53           /*schedule_callback=*/nullptr, std::move(error_callback));
54     }
```

<br/>

コールバックの定義箇所

実際にはGraphOutputStream (OutputStreamObserverの基底クラス) のNotifyとScheduler::EmittedObservedOutputが呼ばれている。
<!-- NOTE-swimm-snippet: the lines below link your snippet to Swimm -->
### 📄 mediapipe/framework/calculator_graph.cc
```c++
776      for (auto& graph_output_stream : graph_output_streams_) {
777        graph_output_stream->PrepareForRun(
778            [&graph_output_stream, this] {
779              absl::Status status = graph_output_stream->Notify();
780              if (!status.ok()) {
781                RecordError(status);
782              }
783              scheduler_.EmittedObservedOutput();
784            },
785            [this](absl::Status status) { RecordError(status); });
786      }
787    
```

<br/>

キューに含まれるパケットそれぞれについてコールバックが実行される
<!-- NOTE-swimm-snippet: the lines below link your snippet to Swimm -->
### 📄 mediapipe/framework/graph_output_stream.cc
```c++
69     absl::Status OutputStreamObserver::Notify() {
70       // Lets one thread perform packets notification as much as possible.
71       // Other threads should quit if a thread is already performing notification.
72       {
73         absl::MutexLock l(&mutex_);
74     
75         if (notifying_ == false) {
76           notifying_ = true;
77         } else {
78           return absl::OkStatus();
79         }
80       }
81       while (true) {
82         bool empty;
83         Timestamp min_timestamp = input_stream_->MinTimestampOrBound(&empty);
84         if (empty) {
85           // Emits an empty packet at timestamp_bound.PreviousAllowedInStream().
86           if (observe_timestamp_bounds_ && min_timestamp < Timestamp::Done()) {
87             Timestamp settled = (min_timestamp == Timestamp::PostStream()
88                                      ? Timestamp::PostStream()
89                                      : min_timestamp.PreviousAllowedInStream());
90             if (last_processed_ts_ < settled) {
91               MP_RETURN_IF_ERROR(packet_callback_(Packet().At(settled)));
92               last_processed_ts_ = settled;
93             }
94           }
95           // Last check to make sure that the min timestamp or bound doesn't change.
96           // If so, flips notifying_ to false to allow any other threads to perform
97           // notification when new packets/timestamp bounds arrive. Otherwise, in
98           // case of the min timestamp or bound getting updated, jumps to the
99           // beginning of the notification loop for a new iteration.
100          {
101            absl::MutexLock l(&mutex_);
102            Timestamp new_min_timestamp =
103                input_stream_->MinTimestampOrBound(&empty);
104            if (new_min_timestamp == min_timestamp) {
105              notifying_ = false;
106              break;
107            } else {
108              continue;
109            }
110          }
111        }
112        int num_packets_dropped = 0;
113        bool stream_is_done = false;
114        Packet packet = input_stream_->PopPacketAtTimestamp(
115            min_timestamp, &num_packets_dropped, &stream_is_done);
116        RET_CHECK_EQ(num_packets_dropped, 0).SetNoLogging()
117            << absl::Substitute("Dropped $0 packet(s) on input stream \"$1\".",
118                                num_packets_dropped, input_stream_->Name());
119        MP_RETURN_IF_ERROR(packet_callback_(packet));
120        last_processed_ts_ = min_timestamp;
121      }
122      return absl::OkStatus();
123    }
124    
```

<br/>

condition variableへのシグナル通知

アプリケーション（CalculatorGraphの呼び出し側）のスレッドで同期するために使われる。
<!-- NOTE-swimm-snippet: the lines below link your snippet to Swimm -->
### 📄 mediapipe/framework/scheduler.cc
```c++
252    void Scheduler::EmittedObservedOutput() {
253      absl::MutexLock lock(&state_mutex_);
254      observed_output_signal_ = true;
255      if (waiting_for_observed_output_) {
256        state_cond_var_.SignalAll();
257      }
258    }
259    
```

<br/>

This file was generated by Swimm. [Click here to view it in the app](/repos/Z2l0aHViJTNBJTNBbWVkaWFwaXBlJTNBJTNBZW1ha3J5bw==/docs/tvr4j).
