const plugin = {
  /************************************
   *str:url地址
   ***********************************/
  openUrl: (str) => {
    window.open(str)
  },
  /************************************
   *data:需要处理的时间戳数据
   *type:type=>Date
   *return:yyyy-MM-dd
   ***********************************/
  getTime: (date) => {
    if (date && !new Date(date)) {
      throw new TypeError('The correct format was not obtained for function "getTime"')
    }
    let nowDate = date ? new Date(date) : new Date()
    const year = nowDate.getFullYear()
    const month = nowDate.getMonth() + 1
    const data = nowDate.getDate()
    return [year, month.toString()[1] ? month : '0' + month, data.toString()[1] ? data : '0' + data].join('-')
  },
  /************************************
   *data:需要深度克隆的数据
   *type:all
   *return:data
   ***********************************/
  cloneData: (data) => {
    let _this = plugin
    let type = _this.getDataType(data)
    let newData = null
    if (type === 'Array') {
      newData = []
      data.forEach((item, index) => {
        newData[index] = _this.cloneData(item)
      })
    } else if (type === 'Object') {
      newData = {}
      for (let index in data) {
        let item = data[index]
        newData[index] = _this.cloneData(item)
      }
    } else {
      newData = data ? JSON.parse(JSON.stringify(data)) : data
    }
    return newData
  },
  /************************************
   *data:需要判断类型的数据
   *type:all
   *return:String|Number|Null|Undefined|Object|Array|Function
   ***********************************/
  getDataType: (data) => {
    if (data === null) {
      return 'Null'
    } else if (data === undefined) {
      return 'Undefined'
    }
    return Object.prototype.toString.call(data).split(' ')[1].split(']')[0]
  },
  /************************************
   *data:需要扁平化的数据
   *type:Object|Array
   *return:Array|Object
   ***********************************/
  flatten: (data) => {
    let _this = plugin
    let oldData = _this.cloneData(data)
    let type = _this.getDataType(oldData)
    if (type === 'Object') {
      for (let index in oldData) {
        let item = oldData[index]
        let itemType = _this.getDataType(item)
        if (itemType === 'Object') {
          let deleteProp = _this.cloneData(item) // 保存一份需要处理的数据
          delete oldData[index]
          return _this.flatten({ ...oldData, ...deleteProp })
        } else if (itemType === 'Array') {
          let newData = []
          let deleteProp = _this.cloneData(item) // 保存一份需要处理的数据
          delete oldData[index]
          if (deleteProp.length < 1) {
            newData.push({ ...oldData })
          } else {
            deleteProp.forEach(itemDel => {
              newData.push({ ...oldData, [index]: itemDel })
            })
          }
          return _this.flatten(newData)
        }
      }
      return oldData
    } else if (type === 'Array') {
      for (let index in oldData) {
        let item = oldData[index]
        let itemType = _this.getDataType(item)
        if (itemType === 'Object') {
          oldData[index] = _this.flatten(item)
        } else if (itemType === 'Array') {
          let newArr = []
          oldData.forEach(itemOld => {
            newArr.push(...itemOld)
          })
          return _this.flatten(newArr)
        }
      }
      return oldData
    } else {
      return oldData
    }
  },

  /************************************
   *data:需要合并的数据
   *rule:合并规则
   *type:Array
   *return:Array
   ***********************************/

  /************************************
   *rule示例
   *{mainRule:'product_id/id',otherRule:[{name:'image/img'},{name:'product_code'},{name:'number',type:'add'}],childrenName:'other_info',childrenRule:{mainRule:['size/size_name','color']}}
   *输出：[
     {
       id:value,
       img:value,
       product_code:value,
       number:value(add),
       other_info:[{
         size_name:value,
         color:value,
         childrenMergeInfo:[{},{}]
       }]
     }
   ]
   *rule参数说明
   *mainRule:主要合并项（根据该项进行合并），当主要合并项为多个是可以为数组；格式：['size','color'] 。“/”在需要改变其key值时添加，“/”前为处理数据中的key,“/”后为处理后输出数据你为其命名的key
   *otherRule:其他处理项，name为处理项的key值,type为特殊处理，当前取值仅可为'add'，是在合并时累加该项，当type不存在时  name项会保留在mainRule同一个数据层级，“/”用法参考mainRule
   *childrenName:命名子合并项key值，默认值childrenMergeInfo，
   *childrenRule:多层级合并时传入，具体规则参考上方
   ***********************************/
  mergeData: (datas, rule) => {
    let _this = plugin
    let data = _this.cloneData(datas)
    let newData = []
    if (_this.getDataType(data) === 'Array') {
      data.forEach(item => {
        let ruleType = _this.getDataType(rule.mainRule)
        let flag = newData.find(value => {
          if (ruleType === 'Array') { // 处理根据多个key合并的情况
            let itemStr = []
            let valueStr = []
            rule.mainRule.forEach(itemMain => {
              let mainName = itemMain.split('/')
              itemStr.push(item[mainName[0]])
              valueStr.push(value[mainName[1] || mainName[0]])
            })
            return itemStr.join('/') === valueStr.join('/')
          } else if (ruleType === 'String') {
            let mainName = rule.mainRule.split('/')
            return value[mainName[1] || mainName[0]] === item[mainName[0]]
          }
        })
        let cloneItem = _this.cloneData(item) // clone其他项数据以方便保留
        if (!flag) {
          let obj = {}
          if (ruleType === 'Array') {
            rule.mainRule.forEach(itemRule => {
              let mainName = itemRule.split('/')
              obj[mainName[1] || mainName[0]] = item[mainName[0]]
              delete cloneItem[mainName[0]]
            })
          } else if (ruleType === 'String') {
            let mainName = rule.mainRule.split('/')
            obj[mainName[1] || mainName[0]] = item[mainName[0]]
            delete cloneItem[mainName[0]]
          }
          if (_this.getDataType(rule.otherRule) === 'Array' && rule.otherRule.length > 0) {
            rule.otherRule.forEach(itemRule => {
              let otherName = itemRule.name.split('/')
              obj[otherName[1] || otherName[0]] = item[otherName[0]]
              delete cloneItem[otherName[0]]
            })
          }
          if (rule.childrenName) {
            obj[rule.childrenName] = [cloneItem]
          } else {
            obj.childrenMergeInfo = [cloneItem]
          }
          newData.push(obj)
        } else {
          if (ruleType === 'Array') {
            rule.mainRule.forEach(itemMain => {
              let mainName = itemMain.split('/')
              delete cloneItem[mainName[0]]
            })
          } else if (ruleType === 'String') {
            let mainName = rule.mainRule.split('/')
            delete cloneItem[mainName[0]]
          }
          if (_this.getDataType(rule.otherRule) === 'Array' && rule.otherRule.length > 0) {
            rule.otherRule.forEach(itemRule => {
              let otherName = itemRule.name.split('/')
              delete cloneItem[otherName[0]]
              if (itemRule.type === 'add') {
                flag[otherName[1] || otherName[0]] = Number(flag[otherName[1] || otherName[0]]) + Number(item[otherName[0]])
              }
            })
          }
          if (rule.childrenName) {
            flag[rule.childrenName].push(cloneItem)
          } else {
            flag.childrenMergeInfo.push(cloneItem)
          }
        }
      })
      if (rule.childrenRule) {
        newData.forEach(item => {
          item[rule.childrenName || 'childrenMergeInfo'] = _this.mergeData(item[rule.childrenName || 'childrenMergeInfo'], rule.childrenRule)
        })
      }
    } else {
      let type = _this.getDataType(data)
      throw new TypeError('An unknown error occurred from the mergeData function, and the parameter "data" expects to get an "array" but gets an "' + type + '"')
    }
    return newData
  },
  /************************************
   *data:需要处理的数据
   *type:Number|String
   *return:Number
   ***********************************/
  toFixedAuto: (number) => {
    if (number % 1 === 0) {
      return parseInt(number)
    } else if (number % 0.1 === 0) {
      return Number(Number(number).toFixed(1))
    } else if (number % 0.01 === 0 || number % 0.01 !== 0) {
      return Number(number.toFixed(2))
    }
  }
}
export default {
  install (Vue) {
    Vue.prototype.$openUrl = plugin.openUrl
    Vue.prototype.$getTime = plugin.getTime
    Vue.prototype.$clone = plugin.cloneData
    Vue.prototype.$flatten = (data) => { return plugin.flatten(plugin.flatten(data)) }
    Vue.prototype.$mergeData = plugin.mergeData
    Vue.prototype.$toFixed = plugin.toFixedAuto
  }
}
