 private void handleCrStatus2New(JsonNode formData) {
        ObjectNode formDataResult = (ObjectNode) formData;
        DataDictDetailMapper dataDictDetailMapper = SpringContextUtils.getBean(DataDictDetailMapper.class);
        DataDictMapper dataDictMapper = SpringContextUtils.getBean(DataDictMapper.class);
        DataDict dataDict = new DataDict();
        dataDict.setIsDel(0);
        dataDict.setDictCode("crStatus");
        DataDict crStatusDataDict = dataDictMapper.selectOneByParam(dataDict);
        List<DataDictDetail> dataDictDetails = dataDictDetailMapper.selectDetailByDictIdAndLevel(crStatusDataDict.getId(), 1, crStatusDataDict.getAccountId());
        Optional<DataDictDetail> first = dataDictDetails.stream().filter(item -> item.getData().equals("New")).findFirst();
        if (first.isPresent()) {
            formDataResult.put("crStatus", first.get().getId());
            formDataResult.put("crStatus_value", "New");
        }
    }
