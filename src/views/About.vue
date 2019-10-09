<template>
  <div class="about">
      <el-table :data="exerciseAry" style="width: 100%" stripe>
        <el-table-column type="index" label="题号" align="center" width="60"></el-table-column>
        <el-table-column prop="content" label="习题"></el-table-column>
        <el-table-column label="答案" width="150" align="center">
            <template slot-scope="scope">
                <span v-show="answerFlag">{{scope.row.answer}}</span>
            </template>
        </el-table-column>
        <el-table-column label="你的答案" width="150" align="center">
            <template slot-scope="scope">
                <el-input v-model="scope.row.userAnswer" placeholder="请输入答案"></el-input>
            </template>
        </el-table-column>
    </el-table>
    <el-dialog :visible.sync="dialogVisible" width="600px" custom-class="dialog" top="25vh" :show-close="false">
        <p>Correct: {{correctAmount}} ({{correctAry.toString()}})</p>
        <p>Wrong: {{wrongAmount}} ({{wrongAry.toString()}})</p>
        <span slot="footer" class="dialog-footer">
            <el-button type="primary" @click="answerFlag = true" class="show">显示答案</el-button>
            <el-button type="info" @click="answerFlag = false"  class="hide">隐藏答案</el-button>
        </span>
    </el-dialog>
    <el-button type="primary" round class="btn" @click="clickSubmit">提交</el-button>
    <el-button type="success" round class="btn" @click="clickSave">保存答案至本地</el-button>
    <el-button type="info" round class="btn" @click="clickUploadExercise">上传题目文件</el-button>
    <el-button type="info" round class="btn" @click="clickUploadUserAnswer">上传你的答案</el-button>
    <input type="file" ref="exercise" style="display: none" @change="uploadExercise">
    <input type="file" ref="userAnswer" style="display: none" @change="uploadUserAnswer">
  </div>
</template>

<script>
export default {
    data() {
        return {
            exerciseAmount: 10,
            exerciseNumberSize: 1,
            exerciseAry: [],
            answerFlag: false,
            dialogVisible: false,
            correctAmount: 0,
            correctAry: [],
            wrongAmount: 0,
            wrongAry: []
        }
    },
    created() {
        //获取传入的参数n、r
        this.exerciseAmount = Number(this.$route.query.n)
        this.exerciseNumberSize = Number(this.$route.query.r)
        this.exerciseAry = this.createExercises(this.exerciseAmount)
    },
    methods: {
        //生成题目集
        createExercises(exerciseAmount){
            let resAry = []
            for(let i=0;i<exerciseAmount;i++) {
                let newExercise = this.completeExercise(this.createBasedExercise())
                //与前面生成好的题目集进行查重检测
                for (let j = 0; j < i; j++) {
                    if(this.isSame(resAry[j], newExercise)){   
                        newExercise = this.completeExercise(this.createBasedExercise())
                        j=0
                    }
                }
                //到了这一步，说明这道题是不重复的
                resAry.push(newExercise)
            }
            return resAry
        },
        //初步随机生成一道题目（没有判断“-”会不会产生负数）
        createBasedExercise(){
            let str = this.createNumber()
            let characterAry = ["+", "-", "×", "÷", "("]
            let characterAmount = Math.floor(Math.random()*3+1)
            for(let i=0;i<characterAmount;i++) {
                let character = str[0]==="("||i===0||i===characterAmount-1 ? characterAry[Math.floor(Math.random()*4)] : characterAry[Math.floor(Math.random()*5)]
                str = character==='(' ? `(${str})` : `${str} ${character} ${this.createNumber()}`
            }
            return str
        },
        //进一步对题目完善（判断“-”会不会产生负数，并将该题目的计算过程和答案保存下来）；如果题目不符合要求就重新生成一道
        completeExercise(str){
            let result = this.getComputedProgress(str)
            if (result===false) {
                return this.completeExercise(this.createBasedExercise())
            } else {
                result.answer = this.formatAnswer(result.answer)
                return {
                    content: str,
                    ...result,
                    userAnswer: ""//数据实例创建之后添加新的属性到数据实例上，不会触发视图更新,所以这里要给用户答案先写上userAnswer属性，形成映射
                }
            }
        },
        //随机生成一个数（自然数或真分数）
        createNumber(){
            let num = Math.floor(Math.random()*(this.exerciseNumberSize)+1)
            if (num===this.exerciseNumberSize) {
                //生成真分数
                let down = Math.floor(Math.random()*(this.exerciseNumberSize-2)+2)
                let up = Math.floor(Math.random()*(down-1)+1)
                return `${up}/${down}`
            } else {
                //生成自然数
                return '' + num
            }
        },
        //判断该字符串是一个数还是一个算式（0：自然数，1：分数，2：算式）
        isNumber(str){
            if (!isNaN(str)) return 0
            if (/^\d+\/\d+$/.test(str)) return 1
            return 2
        },
        //基本运算（加减乘除以及真分数的运算），返回false（运算产生负数导致false）或运算结果
        compute(str){
            str = str.replace(/\s+/g, "")
            let character = /[+|\-|×|÷]/.exec(str)[0]
            let num1 = str.split(character)[0]
            let num2 = str.split(character)[1]
            character = character==='×' ? '*' : character==='÷' ? '/': character
            if (this.isNumber(num1)===0 && this.isNumber(num2)===0) {
                //自然数与自然数的运算
                num1 = Number(num1)
                num2 = Number(num2)
                switch(character){
                    case '-':
                        return num1-num2>=0 ? num1-num2 : false
                    case '/':
                        return num1%num2===0 ? num1/num2 : `${num1}/${num2}`
                    default:
                        return eval(`${num1}${character}${num2}`)
                }
            }
            if (this.isNumber(num1)===0 && this.isNumber(num2)===1) {
                //自然数与真分数的运算
                num1 = Number(num1)
                let up = Number(num2.split('/')[0])
                let down = Number(num2.split('/')[1])
                switch(character){
                    case '+':
                        return `${num1*down+up}/${down}`
                    case '-':
                        return num1*down-up>=0 ? `${num1*down-up}/${down}` : false
                    case '*':
                        return `${num1*up}/${down}`
                    case '/':
                        return `${num1*down}/${up}`
                }
            }
            if (this.isNumber(num1)===1 && this.isNumber(num2)===0) {
                //真分数与自然数的运算
                let up = Number(num1.split('/')[0])
                let down = Number(num1.split('/')[1])
                num2 = Number(num2)
                switch(character){
                    case '+':
                        return `${num2*down+up}/${down}`
                    case '-':
                        return up-num2*down>=0 ? `${up-num2*down}/${down}` : false
                    case '*':
                        return `${num2*up}/${down}`
                    case '/':
                        return `${up}/${num2*down}`
                }
            }
            if (this.isNumber(num1)===1 && this.isNumber(num2)===1) {
                //真分数与真分数的运算
                let up1 = Number(num1.split('/')[0])
                let down1 = Number(num1.split('/')[1])
                let up2 = Number(num2.split('/')[0])
                let down2 = Number(num2.split('/')[1])
                switch(character){
                    case '+':
                        return `${up1*down2+up2*down1}/${down1*down2}`
                    case '-':
                        return up1*down2-up2*down1>=0 ? `${up1*down2-up2*down1}/${down1*down2}` : false
                    case '*':
                        return `${up1*up2}/${down1*down2}`
                    case '/':
                        return `${up1*down2}/${down1*up2}`
                }
            }
        },
        //获取基础算式（不含括号的算式）的计算过程，返回false或一个对象（包含计算过程和答案）
        getBasedComputedProgress(str){
            let flag = true
            let computedProgress = []
            str = str.replace(/\s+/g, "")
            while(flag && this.isNumber(str)===2) {
                if (str.indexOf("×")>-1 || str.indexOf("÷")>-1) {
                    //有乘除运算，先算乘除
                    str = str.replace(/(\d+)?(\d+\/\d+)?[×|÷](\d+\/\d+)?(\d+)?/, (...arg)=>{
                        let res = this.compute(arg[0])
                        if(res===false) {
                            flag = false
                            return ""
                        } else {
                            computedProgress.push(arg[0])
                            return res
                        }
                    })    
                } else {
                    //没乘除运算，算加减
                    str = str.replace(/(\d+)?(\d+\/\d+)?[+|-](\d+\/\d+)?(\d+)?/, (...arg)=>{
                        let res = this.compute(arg[0])
                        if(res===false) {
                            flag = false
                            return ""   
                        } else {
                            computedProgress.push(arg[0])
                            return res
                        }
                    }) 
                }
            }
            if (!flag) {
                return false
            } else {
                return {
                    answer: str,
                    computedProgress
                }
            }
        },
        //获取算式（包含括号）的计算过程，返回false或一个对象（包含计算过程和答案）
        getComputedProgress(str){
            let progress = {
                answer: null,
                computedProgress: []
            }
            str = str.replace(/\s+/g, "")
            if(str.indexOf('(')>-1){
                //包含括号，利用栈处理
                let stack = [] 
                //把式子的每一个字符都压入栈
                for (let i = 0; i < str.length; i++) {
                    const char = str[i];
                    if (char!==')') {
                        stack.push(char)
                    } else {
                        //遇到‘）’，就弹出整个括号内的算式
                        let popChar = ""
                        let basedStr = ""
                        while((popChar = stack.pop(char))!=="("){
                            basedStr= popChar + basedStr
                        }
                        let basedProgress = this.getBasedComputedProgress(basedStr)
                        if (basedProgress===false) return false
                        stack.push(basedProgress.answer)
                        progress.computedProgress = progress.computedProgress.concat(basedProgress.computedProgress)
                    }
                }
                //最后栈内还会剩余一个算式，对其进行基础算式处理
                let stackProgress = this.getBasedComputedProgress(stack.join(""))
                if (stackProgress===false) return false
                progress.answer = stackProgress.answer
                progress.computedProgress = progress.computedProgress.concat(stackProgress.computedProgress)
            } else {
                //不含括号，直接按基础算式求得结果
                progress = this.getBasedComputedProgress(str)
            }
            return progress
        },
        //格式化答案
        formatAnswer(str){
            if (this.isNumber(str)===1) {
                let up = Number(str.split('/')[0])
                let down = Number(str.split('/')[1])
                if (up>down) {
                    let mcd = this.maxCommonDivisor(up, down)
                    up = up/mcd
                    down = down/mcd
                    str = down===1 ? `${up}` : `${Math.floor(up/down)}'${up%down}/${down}`
                } else if(up===down){
                    str = '1'
                } else {
                    let mcd = this.maxCommonDivisor(down, up)
                    up = up/mcd
                    down = down/mcd
                    str = `${up}/${down}`
                }
            }
            return str
        },
        //求最大公约数
        maxCommonDivisor(a, b){
            let c=b
            while(b!==0){
                c=a%b
                a=b
                b=c
            }
            return a
        },
        //判断两题目是否相同（通过比较两算式的运算过程检查查复），如果相同则返回true
        isSame(exercise1, exercise2){
            //由于运算符号最多3个，所以运算过程的数组最多只有3项
            if (exercise1.answer!==exercise2.answer)    return false
            let progress1 = exercise1.computedProgress.slice()
            let progress2 = exercise2.computedProgress.slice()
            if (progress1.length!==progress2.length)    return false
            if (progress1.length===3) {
                //如果运算过程有3项，需要考虑一种运算过程相似的情况，例如：["3×1", "2×6", "3+12"]和["2×6", "3×1", "12+3"]不完全相同，但这两道题是相同的
                let similar = this.isSimilar(progress1.pop(), progress2.pop())
                if(similar===2) return false
                if (similar===1) {
                    [progress1[0], progress1[1]] = [progress1[1], progress1[0]]
                }
                while (progress1.length) {
                    if(this.isSimilar(progress1.pop(), progress2.pop())!==0) return false
                }
                return true
            } else {
                //如果运算过程不是3项，那么每一项运算过程都必须完全相同，不能相似
                while (progress1.length) {
                    if(this.isSimilar(progress1.pop(), progress2.pop())!==0) return false  
                }
                return true
            }
        },
        //比较两算式是相似还是相同还是不同，0：相同，1：相似，2：不同（3+1和3+1是相同，3+1和1+3是相似）
        isSimilar(str1, str2){
            let character1 = /([+|\-|×|÷])/.exec(str1)[0]
            let character2 = /([+|\-|×|÷])/.exec(str2)[0]
            if (character1!==character2)    return 2
            let ary1 = str1.split(character1)
            let ary2 = str1.split(character1)
            if (ary1[0]===ary2[0] && ary1[1]===ary2[1]) return 0
            if (ary1[0]===ary2[1] && ary1[1]===ary2[0] && (character1==='+'||character1==='×')) return 1
            return 2
        },
        //点击提交按钮
        clickSubmit(){
            this.correctAmount = 0
            this.correctAry = []
            this.wrongAmount = 0
            this.wrongAry = []
            this.exerciseAry.forEach((item,index) => {
                if (item.answer===item.userAnswer) {
                    this.correctAmount++
                    this.correctAry.push(index+1)
                } else {
                    this.wrongAmount++
                    this.wrongAry.push(index+1)
                }
            });
            this.dialogVisible = true
        },
        //点击保存答案文件
        clickSave(){
            let content = ""
            this.exerciseAry.forEach((item, index)=>{
                content += `${index+1}. ${item.answer}\n`
            })
            let blob = new Blob([content], {type: "text/plain;charset=utf-8"});
            saveAs(blob, "answer.txt");
        },
        //点击上传本地题目文件按钮
        clickUploadExercise(){
            this.$refs["exercise"].click()
        },
        //上传本地题目文件
        uploadExercise(){
            let file = this.$refs["exercise"].files[0]
            let reader = new FileReader()
            let _this = this
            reader.readAsText(file);
            reader.onload = function () {
                let ary = this.result.split("\n")
                //将题目存进一个数组
                ary = ary.map(item=>{
                    if (item.indexOf('.')>-1) {
                        return item.slice(item.indexOf('.')+1).trim()
                    } else {
                        return item.trim()
                    }
                })
                //将这些题目完善（加入答案和运算过程），形成一个新的数组
                let resAry = []
                for(let i=0;i<ary.length;i++) {
                    let newExercise = _this.completeExercise(ary[i])
                    //与前面生成好的题目集进行查重检测
                    for (let j = 0; j < i; j++) {
                        if(_this.isSame(resAry[j], newExercise)){   
                            newExercise = _this.completeExercise(_this.createBasedExercise())
                            j=0
                        }
                    }
                    //到了这一步，说明这道题是不重复的
                    resAry.push(newExercise)
                }
                //将新数组赋给exerciseAry
                _this.exerciseAry = resAry
            }
        },
        //点击上传用户答案文件按钮
        clickUploadUserAnswer(){
            this.$refs["userAnswer"].click()
        },
        //上传用户答案文件
        uploadUserAnswer(){
            let file = this.$refs["userAnswer"].files[0]
            let reader = new FileReader()
            let _this = this
            reader.readAsText(file);
            reader.onload = function () {
                let ary = this.result.split("\n")
                //将用户答案存进一个数组
                ary = ary.forEach((item,index)=>{
                    if (item.indexOf('.')>-1) {
                        _this.exerciseAry[index].userAnswer = item.slice(item.indexOf('.')+1).trim()
                    } else {
                        _this.exerciseAry[index].userAnswer = item.trim()
                    }
                })
                console.log(_this.exerciseAry)
            }
        }
    }
}
</script>

<style scoped>
.about {
    margin: 10px auto 20px auto;
    width: 600px;
}
.about .btn{
    margin: 20px 0 0 0;
    width: 600px;
}
.about .dialog p{
    margin: 20px 90px;
    font-size: 24px;
}
.about .dialog .dialog-footer {
    display: block;
    padding: 0 90px 40px 90px;
}
.about .dialog .dialog-footer .show{
    float: left;
    width: 150px;
}
.about .dialog .dialog-footer .hide{
    width: 150px;
}
</style>