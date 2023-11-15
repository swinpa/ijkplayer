##IJKPlayer



```
IJKPlayer也是一个知名的开源视频播放器，是基于FFmpeg和SDL2的跨平台播放器。它支持多种视频格式和协议，包括RTMP、HLS、HTTP等，
可以用于iOS、Android等平台的应用开发。IJKPlayer的优点包括播放稳定性、适应性强、支持多种视频格式和协议等。同时，它也有一个活
跃的社区和开发团队，不断地更新和改进这个开源项目。因此，如果您需要一个高效、稳定的视频播放器，IJKPlayer也是一个不错的选择。

相较于前面提到的几个开源视频播放器，IJKPlayer有以下一些优点和缺点：

优点：

1. 支持多种视频格式和协议，包括RTMP、HLS、HTTP等，能够适应不同的应用场景。
2. 播放稳定性高，能够保证视频播放的流畅和稳定性。
3. 适应性强，能够根据不同的设备和网络条件，自动调整视频的清晰度和码率。
4. 支持硬件解码，能够利用设备的硬件资源来提高播放效率。
5. 社区活跃，有大量的用户和开发者参与其中，能够提供技术支持和改进建议。

缺点：

1. 作为一个底层的播放器库，使用起来需要一定的开发经验和技术能力，对于非开发人员来说可能不太友好。
2. 在特定的场景下可能会存在兼容性问题，需要开发者自行解决。
3. 由于IJKPlayer的使用是基于FFmpeg和SDL2的，因此需要遵守相关的开源协议，不能随意使用和修改代码。

```


###框架内核流程
1. 业务外部通过IJKFFMoviePlayerController.init(contentURL: url, with: options)创建播放器实例，然后调用IJKFFMoviePlayerController.prepareToPlay()开始了播放器的视频文件读取，解码渲染等操作
	
	1. 在经历prepareToPlay --> ijkmp_prepare_async --> ijkmp_prepare_async_l --> ffp_prepare_async_l --> stream_open后
	
	 最终在stream_open中调用is->video_refresh_tid = SDL_CreateThreadEx(&is->_video_refresh_tid, video_refresh_thread, ffp, "ff_vout");
开辟子线程（线程入口函数为video_refresh_thread）不断地读取帧队列中的视频数据进行渲染
	
* 在stream_open中，在创建渲染线程（video_refresh_thread）后会接着创建数据读取线程（read_thread）不断地从磁盘或网络中读取视频数据
	通过is->read_tid = SDL_CreateThreadEx(&is->_read_tid, read_thread, ffp, "ff_read");创建视频数据读取线程，线程入口函数是read_thread()
	ff_ffplay.cread_thread(){
		for (;;) {
			//读取数据流(stream)中的数据，放入到队列中
		}
	}
	
* ff_ffplay.c --> video_refresh()  /* called to display each frame */

	该方法从帧队列（FrameQueue）中获取一帧视频数据，然后调用video_display2开始进行渲染流程
	1. 最终会调用到ijk_vout_ios_gles2.m --> vout_display_overlay_l() 方法，使用OpenGL进行渲染
	2. 大概的调用栈 

			video_refresh_thread() 渲染的线程入口函数，在该线程函数中进行
			while (true) {
				video_refresh()
			}
			--> video_refresh()
				--> video_display2() 
					--> video_image_display2() 
						--> SDL_VoutDisplayYUVOverlay()
							--> vout->display_overlay(vout, overlay) ( display_overlay == vout_display_overlay)
								--> vout_display_overlay_l() (内部进行OpenGL渲染)
								
								
##v-2

* 视频解码前期准备调用栈

	```
	- (id)initWithContentURLString:(NSString *)aUrlString
                   withOptions:(IJKFFOptions *)options
                   
       -> ijkmp_ios_create -> ffpipeline_create_from_ios
       
       在ffpipeline_create_from_ios  中会设置创建解码器的函数入口 
       pipeline->func_open_video_decoder = func_open_video_decoder;
       当需要解码时会调用该函数创建解码器（在read_thread -> stream_component_open -> ffpipeline_open_video_decoder -> func_open_video_decoder时创建，然后再）
                   
	```

* 视频解码调用栈

	```
	read_thread -> stream_component_open -> ffpipeline_open_video_decoder -> func_open_video_decoder -> ffpipenode_create_video_decoder_from_ios_videotoolbox -> ijk_VideoToolbox_sync_Create -> ijk_VideoToolbox_CreateInternal -> videotoolbox_sync_create
	
	最终会在videotoolbox_sync_create中调用VTDecompressionSessionCreate() 方法创建用于视频解码的session
	并指定了解码回调方法为VTDecoderCallback
	
	```

* 视频循环解码调用栈

	```
	video_thread -> ffpipenode_run_sync -> videotoolbox_sync_decode_frame -> decode_video -> decode_video_internal -> VTDecompressionSessionDecodeFrame
	
	```
	
* 视频解密调用栈

	```
	
	read_thread -> av_read_frame -> read_frame_internal -> ff_read_packet -> mov_read_packet -> append_packet_chunked -> avio_read -> retry_transfer_wrapper -> ijkio_read -> ijkio_manager_io_read -> decrypt_read ->  IOS2ByteSource::read -> MediaDataSource::read -> [HttpDataSource ocRead::]  -> [HttpDataSource getDataWithStartOffset:readLength:] 
	
	#14	0x0000000104586e48 in read_thread at /Users/wp/Documents/StudySrc/IJKPlayer_Learning/IJKMediaPlayer-FLT/ijkmedia/ijkplayer/ff_ffplay.c:3549
	#13	0x0000000104677310 in av_read_frame ()
	#12	0x0000000104677584 in read_frame_internal at /Users/hzc/Documents/ijkplayer-ex/ios/ffmpeg-arm64/libavformat/utils.c:1591
	#11	0x00000001046768bc in ff_read_packet at /Users/hzc/Documents/ijkplayer-ex/ios/ffmpeg-arm64/libavformat/utils.c:866
	#10	0x000000010463fc0c in mov_read_packet at /Users/hzc/Documents/ijkplayer-ex/ios/ffmpeg-arm64/libavformat/mov.c:7118
	#9	0x0000000104675e80 in append_packet_chunked at /Users/hzc/Documents/ijkplayer-ex/ios/ffmpeg-arm64/libavformat/utils.c:293
	#8	0x000000010461244c in avio_read at /Users/hzc/Documents/ijkplayer-ex/ios/ffmpeg-arm64/libavformat/aviobuf.c:657
	#7	0x0000000104610d6c in retry_transfer_wrapper at /Users/hzc/Documents/ijkplayer-ex/ios/ffmpeg-arm64/libavformat/avio.c:376
	#6	0x00000001045694f8 in ijkio_read at /Users/wp/Documents/StudySrc/IJKPlayer_Learning/IJKMediaPlayer-FLT/ijkmedia/ijkplayer/ijkavformat/ijkio.c:84
	#5	0x0000000104599580 in ijkio_manager_io_read at /Users/wp/Documents/StudySrc/IJKPlayer_Learning/IJKMediaPlayer-FLT/ijkmedia/ijkplayer/ijkavformat/ijkiomanager.c:462
	#4	0x00000001045aa1f0 in ::decrypt_read(IjkURLContext *, unsigned char *, int) at /Users/wp/Documents/StudySrc/IJKPlayer_Learning/IJKMediaPlayer-FLT/ijkmedia/ijkplayer/ijkavformat/decrypt/decrypt_for_ffmpeg.cpp:42
	#3	0x000000010456a628 in IOS2ByteSource::read(void*, long long) at /Users/wp/Documents/StudySrc/IJKPlayer_Learning/IJKMediaPlayer-FLT/ijkmedia/ijkplayer/ijkavformat/decrypt/glueDataSource.cpp:13
	#2	0x00000001045c19ec in MediaDataSource::read(void*, long long) at /Users/wp/Documents/StudySrc/IJKPlayer_Learning/IJKMediaPlayer-FLT/IJKMediaPlayer/datasource/HttpDatasource.mm:122
	#1	0x00000001045c1b58 in -[HttpDataSource ocRead::] at /Users/wp/Documents/StudySrc/IJKPlayer_Learning/IJKMediaPlayer-FLT/IJKMediaPlayer/datasource/HttpDatasource.mm:211
	#0	0x00000001045c2034 in -[HttpDataSource getDataWithStartOffset:readLength:] at /Users/wp/Documents/StudySrc/IJKPlayer_Learning/IJKMediaPlayer-FLT/IJKMediaPlayer/datasource/HttpDatasource.mm:301


	```

* 视频数据读取逻辑

	####read_thread读取线程入口函数，读取逻辑如下：
	
	1. 调用ffmpeg 的接口 avformat_open_input 打开一个读取流
	
		在init_input(s, filename, &tmp)中调用AVFormatContext->io_open(也就是io_open_default)
		
		```
		init_input() 
		-> io_open_default() 
			-> ffio_open_whitelist()
				-> ffurl_open_whitelist() （通过调用ffurl_alloc生成URLContext，
										并根据URL找到对应的URLProtocol，后续会在ffurl_connect中调
										用URLProtocol.url_open2()进行网络链接）
					-> ffurl_connect()
						-> URLProtocol.url_open2()
						
		
		const URLProtocol ff_librtmp_protocol = {
		    .name                = "rtmp",
		    .url_open            = rtmp_open,
		    .url_read            = rtmp_read,
		    .url_write           = rtmp_write,
		    .url_close           = rtmp_close,
		    .url_read_pause      = rtmp_read_pause,
		    .url_read_seek       = rtmp_read_seek,
		    .url_get_file_handle = rtmp_get_file_handle,
		    .priv_data_size      = sizeof(LibRTMPContext),
		    .priv_data_class     = &librtmp_class,
		    .flags               = URL_PROTOCOL_FLAG_NETWORK,
		};
		```
		
		1. 最终通过ff_librtmp_protocol->url_open调用到了rtmp_open方法，在rtmp_open中调用librtmp的接口发起RTMP链接请求
		2. 当io_open_default打开流完成（RTMP链接成功）后会调用av_probe_input_buffer2进行音视频流封装格式的打探（通过
		打分的方式确定当前音视频流是那种封装格式）
			
			在这一步中通过avio_read->AVIOContext->read_packet == io_read_packet->ffurl_read->URLProtocol->url_read==rtmp_read
			调用到了librtmp的读取接口读取一定长度的数据，然后
			av_probe_input_format2->av_probe_input_format3->AVInputFormat->read_probe分析数据流是否是对应的格式
			在FFMPEG中全局保存了所有支持的format
			
			
			```
			static const AVInputFormat * const demuxer_list[] = {
		     &ff_aac_demuxer,
		     &ff_concat_demuxer,
		     &ff_data_demuxer,
		     &ff_flac_demuxer,
		     &ff_flv_demuxer,
		     &ff_live_flv_demuxer,
		     &ff_hevc_demuxer,
		     &ff_hls_demuxer,
		     &ff_matroska_demuxer,
		     &ff_mov_demuxer,
		     &ff_mp3_demuxer,
		     &ff_mpegps_demuxer,
		     &ff_mpegts_demuxer,
		     &ff_mpegvideo_demuxer,
		     &ff_webm_dash_manifest_demuxer,
		     &ff_ijklivehook_demuxer,
		     &ff_ijklas_demuxer,
		     NULL };
		
		     AVInputFormat ff_flv_demuxer = {
		         .name           = "flv",
		         .long_name      = NULL_IF_CONFIG_SMALL("FLV (Flash Video)"),
		         .priv_data_size = sizeof(FLVContext),
		         .read_probe     = flv_probe,
		         .read_header    = flv_read_header,
		         .read_packet    = flv_read_packet,
		         .read_seek      = flv_read_seek,
		         .read_close     = flv_read_close,
		         .extensions     = "flv",
		         .priv_class     = &flv_class,
		     };
			```
		3. 确定了封装格式后，就根据封装格式读取头部信息，分析出音视频文件有几条流
		4. 通过以上两部已经完成了RTMP的链接，以及确认了音视频流的封装格式，以及有几条流，接下来就可以不断的读取音视频流
	2. 开启一个for循环，在循环内部调用 ffmpeg 的接口av_read_frame 不停的读取数据流的每一帧
	
	av_read_frame->read_frame_internal->ff_read_packet->AVInputFormat->read_packet-> flv_read_packet->av_get_packet->append_packet_chunked->avio_read->ff_librtmp_protocol.url_read==rtmp_read
	
		```
		当然在读取下一帧之前会判断：
		1. 当前播放有没有退出
		2. 当前播放有没有停止
		3. 当前存放视频帧的队列是否已满
		4. 当前是否已读取结束，读取结束需不需要循环播放
		```
			
	3. 读取成功后把数据放入对应的队列中

* ijk 与FFMPEG交互[参考](https://www.pudn.com/news/6228d6cc9ddf223e1ad2037d.html)

```
read_thread -> avformat_open_input -> init_input -> ffio_open_whitelist -> ffurl_open_whitelist -> ffurl_connect -> ijkio_open

1. 在avformat_open_input 中会调用avformat_alloc_context 来构建AVFormatContext s，并设置s->io_open  = io_open_default;
2. 调用init_input(s, filename, &tmp)

	在init_input 中通过s->io_open(s, &s->pb, filename, AVIO_FLAG_READ | s->avio_flags, options) 调用上一步(第1步)中的io_open_default
		
		1. 在io_open_default 中调用ffio_open_whitelist(pb, url, flags, &s->interrupt_callback, options, s->protocol_whitelist, s->protocol_blacklist);
		2. 中ffio_open_whitelist()又直接调用ffurl_open_whitelist(&h, filename, flags, int_cb, options, whitelist, blacklist, NULL);
		3. ffurl_open_whitelist 中会先构建URLContext puc，然后执行ffurl_connect(*puc, options);
		
			1. 在ffurl_open_whitelist 会通过puc  = ffurl_alloc(puc, filename, flags, int_cb); 构造URLContext
				在构建过程中会通过url_find_protocol(filename); 从全局url_protocols[] 列表中找到最合适读取filename
				的URLProtocol（ijk 在FFMPEG中扩展的ijk 协议）设置到URLContext->prot 中
		      		static const URLProtocol * const url_protocols[] = {
				    &ff_async_protocol,
				    &ff_cache_protocol,
				    &ff_data_protocol,
				    &ff_ffrtmphttp_protocol,
				    &ff_file_protocol,
				    &ff_ftp_protocol,
				    &ff_hls_protocol,
				    &ff_http_protocol,
				    &ff_httpproxy_protocol,
				    &ff_https_protocol,
				    /*
				     ijkplayer 在这里扩展了ijk 的协议
				     */
				    &ff_ijkhttphook_protocol,
				    &ff_ijklongurl_protocol,
				    &ff_ijkmediadatasource_protocol,
				    &ff_ijksegment_protocol,
				    &ff_ijktcphook_protocol,
				    &ff_ijkio_protocol,
				    &ff_pipe_protocol,
				    &ff_prompeg_protocol,
				    &ff_rtmp_protocol,
				    &ff_rtmpt_protocol,
				    &ff_tee_protocol,
				    &ff_tcp_protocol,
				    &ff_tls_protocol,
				    &ff_udp_protocol,
				    &ff_udplite_protocol,
				    NULL };
		4. 在ffurl_connect中调用puc->prot->url_open2(puc, puc->filename, puc->flags, options) 就能调用到ijk 扩展协议(ff_ijkio_protocol)中的ijkio_open
			
			首先ijk在ffmpeg源码的libavformat->ijkutils.c 中扩展定义了
			URLProtocol ff_ijkio_protocol = {
			     .name                = ijkio,
			     .url_open2           = ijkdummy_open,
			     .priv_data_size      = 1,
			     .priv_data_class     = &ijk_ijkio_context_class,
			 };
			 ijkdummy_open 是一个空方法
			 关键在于ijk，在 void ijkav_register_all(void) 中通过IJK_REGISTER_PROTOCOL(ijkio);
			 使用内存拷贝方式将在ijk中定义的
			 URLProtocol ijkimp_ff_ijkio_protocol = {
			    .name                = "ijkio",
			    .url_open2           = ijkio_open,
			    .url_read            = ijkio_read,
			    .url_seek            = ijkio_seek,
			    .url_close           = ijkio_close,
			    .priv_data_size      = sizeof(Context),
			    .priv_data_class     = &ijkio_context_class,
			 };
			 的内存覆盖到了ff_ijkio_protocol，
			 这就达到了ff_ijkio_protocol的内容实际指向外部ijkimp_ff_ijkio_protocol
			 的内容，所以ff_ijkio_protocol->url_open2 最终就为ijkio_open
     
     int ffurl_open_whitelist(URLContext **puc, const char *filename, int flags, const AVIOInterruptCB *int_cb, AVDictionary **options,
                         const char *whitelist, const char* blacklist, URLContext *parent)

```


