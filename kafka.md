# kafka+Quartz项目中的应用场景

-  配置在disconf上面，使用@DisconfFile（filename=“”，name=“”）注解将配置读取。
-  目前每天2点53分30秒( 30 53 02)的时候会将将所有kafka的消费者unRegister掉（），在这之前会定时（Spring scheduling + Quartz）执行清理任务（DDS，OMCS,单元区域，网点区域）的数据，并且注册kafka消息监听消息并且入库。此后开始散货和集货的数据的数据清洗。