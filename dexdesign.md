

## DexAPI

### dex_start
  @backend_translation_module
  @backend_host
  @backend_port
  @backend_user
  ...

### dex_sql
  @input_list   e.g. list of relnames
  @query
  @output_list (optional)  relname
  returns
  @output: DexDFRef

### dex_get_scanner
  @dex_df: DexDFRef
  @format
  returns
  RowIterator


## DexQueryTranslation

### dex_translate     [Interface]
  @dex_query
  returns
  @dex_request: DexRequest


## DexQueryExecution

### dex_

### dex_send
  @request: DexRequest
  returns
  @reply: DexReply

### dex_send_async
  @request
  return

### dex_get_result     
  @request_id
  return
  @reply: DexReply   //blocking

### DexRequest
  @session
  @request_id
  @inputlist
  @outputlist
  @command

### DexReply
  @request_id
  @result



## DexDataAccess

### dex_get_scanner
  @dfref: DexDFRef
  return
  @it: RowIterator

### dex_get_updator
  @dfref: DexDfref
  return
  ...


-------
Backend
-------

## DexBackendServer

## DexRequestServer

### newInstance

### deserializeRequest
   @reqstr: String
   return
   @request: DexRequest

### createJobs
  @request
  returns
  @jobs: list of jobs

## DexBackendRunner

### submitJobs
  @jobs

### getJobStatus
  @jobid

