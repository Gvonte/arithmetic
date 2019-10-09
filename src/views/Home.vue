<template>
  <div class="home">
    <h1>自动生成小学四则运算题目</h1>
    <h2>请输入您的命令行</h2>
    <div class="input">
      <el-input type="text" v-model="command" placeholder="请输入内容" @keyup.enter.native="goToAbout"></el-input>
    </div>
  </div>
</template>

<script>
export default {
  name: 'home',
  data(){
    return{
      command:""
    }
  },
  methods:{
    goToAbout(){
      let str = this.command
      let reg1 = /^Gvonte\.exe.+-n\s+(\d+)/
      let reg2 = /^Gvonte\.exe.+-r\s+(\d+)/
      //获取传入的参数n、r
      let n = reg1.exec(str)
      let r = reg2.exec(str)
      if (r===null) {
        this.$message('请按照指定命令格式传参，r为必传参数');
      } else if (Number(r[1])<1) {
        this.$message('r参数必须大于等于1');
      } else {
        r = Number(r[1])
        n = n===null ? 10 : Number(n[1])
        this.$router.push(`/about?n=${n}&r=${r}`)
      }
    }
  }
}
</script>

<style scoped>
.home{
  margin: 80px auto;
  width: 600px;
}
.home h1 {
  text-align: center
}
.home h2 {
  margin-top: 40px;
  text-align: center
}
.home .input {
  margin: 50px auto;
  width: 600px;
}
</style>