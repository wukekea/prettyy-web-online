<script setup>
import { ref, computed } from 'vue';
import { getIdentityCodeByEmailAPI} from '@/apis/base.js';
import { useCountDown, countDown } from '@/utils/countDown';
import {ElMessage} from "element-plus";
import { useRouter } from "vue-router";
import {useUserStore} from "@/stores/user.js";
import { useCaptchaStore } from "@/stores/captcha.js";

const userStore = useUserStore()
const captchaStore = useCaptchaStore()

// 控制登录的弹窗是否显示
const showLoginDialog = ref(false)
defineExpose({showLoginDialog})

// 默认显示免密登录的tab
const activeName = ref('1')

// 免密登录表单校验(邮箱和验证码)
// 用户名和验证码只需要简单的配置(看文档的方式 - 复杂的功能通过多个不同的组件拆解)
// 同意协议 自定义校验规则 validator:(rule, value, callback)=>{}
// 统一校验 通过表单form实例的方法 validate -> true
// 1、表单对象
const form1 = ref({
  email: 'yswang837@gmail.com',
  identifyCode1: '',
  method: '',
  agree: true
})
// 2、规则对象
const rules1 = {
  email: [
    {required: true, message: '请输入邮箱', trigger: 'blur'},
    {pattern: /^[a-zA-Z0-9_.+-]+@[a-zA-Z0-9-]+\.[a-zA-Z0-9-.]+$/, message: '邮箱格式不合法', trigger: 'blur'}
  ],
  identifyCode1: [
    {required: true, message: '请输入验证码', trigger: 'blur'},
    {min:6, max:6, message: '验证码必须为6位', trigger: 'blur'},
  ],
  // 自定义校验规则，勾选了checkbox就通过，不勾选就不通过
  agree: [
    {
      // rule 规则，代表这个自定义校验规则是检测的agree字段
      // value 当前checkbox拿到的值
      // callback 回调函数
      validator: (rule, value, callback)=>{
        // console.log('value....',value)
        // console.log('rule',rule)
        if (value) {
          // 校验通过
          callback()
        }else {
          // 校验不通过
          callback(new  Error('请勾选协议'))
        }
      }
    }
  ]
}

// 为了防止用户一上来就点击登录，导致我们写的rules都被绕过了，所以需要拿到form表单的实例，然后调用validate方法
const formRef1 = ref(null)
const router = useRouter()
// 通过邮箱和验证码登录
const loginOrRegisterByCode = async (email, identifyCode1, activeName) => {
  formRef1.value.validate(async (valid) => {
    // 所有项都通过校验，valid才为true
    // console.log(valid)
    if (valid) {
      if (formatTime.value <= 0) {
        ElMessage({type:'info', message: '验证码已过期，请重新获取验证码'})
        return
      }
      await userStore.loginByCodeAndSetUserInfo(email, identifyCode1, activeName)
      ElMessage({type:'success', message: '登录成功'})
      // 跳转到首页
      showLoginDialog.value = false
      await router.push({path: '/'})
    }
  })
}

// 通过账密登录
const formRef2 = ref(null)
const loginOrRegisterByPwd = async (email, password, activeName, identifyId, identifyCode) => {
  formRef2.value.validate(async (valid) => {
    // 所有项都通过校验，valid才为true
    // console.log(valid)
    if (valid) {
      await userStore.loginByPwdAndSetUserInfo(email, password, activeName, identifyId, identifyCode)
      ElMessage({type:'success', message: '登录成功'})
      // 跳转到首页
      showLoginDialog.value = false
      await router.push({path: '/'})
    }
  })

}

// 账密登录表单校验(邮箱，密码和验证码)
// 1、表单对象
const form2 = ref({
  email: 'yswang837@gmail.com',
  password: '123456',
  captcha: '',
  method: '',
  agree: true
})
// 2、规则对象
const rules2 = {
  email: [
    {required: true, message: '请输入邮箱', trigger: 'blur'},
    {pattern: /^[a-zA-Z0-9_.+-]+@[a-zA-Z0-9-]+\.[a-zA-Z0-9-.]+$/, message: '邮箱格式不合法', trigger: 'blur'}
  ],
  password: [
    {required: true, message: '请输入密码', trigger: 'blur'},
    {min:6, max:20, message: '密码的长度为6~20位', trigger: 'blur'},
  ],
  captcha: [
    {required: true, message: '请输入验证码', trigger: 'blur'},
    {min:6, max:6, message: '验证码必须为6位', trigger: 'blur'},
  ],
  // 自定义校验规则，勾选了checkbox就通过，不勾选就不通过
  agree: [
    {
      // rule 规则，代表这个自定义校验规则是检测的agree字段
      // value 当前checkbox拿到的值
      // callback 回调函数
      validator: (rule, value, callback)=>{
        // console.log('value....',value)
        // console.log('rule',rule)
        if (value) {
          // 校验通过
          callback()
        }else {
          // 校验不通过
          callback(new  Error('请勾选协议'))
        }
      }
    }
  ]
}
// 3、获取账密登录时的验证码

const countDown2 = useCountDown()

// 设置5分钟倒计时(如果刷新页面，验证码直接清零，直接过期，因为清除了定时器
// 正常操作 验证码5分钟有效，减少一次请求后端的次数)
//
// 设置60秒的获取验证码按钮，刷新不能直接清除定时器，需要维护localStorage，这是存在客户端的，也容易被篡改。
// 所以在此基础上，继续在后端新增频次限制

// 通过邮箱获取验证码
const { formatTime, start } = countDown(300)
const getIdentityCodeByEmail = async (email)=>{
  await getIdentityCodeByEmailAPI(email)
  ElMessage({type: 'success', message: '验证码已发送，请查看邮箱'})
  start()
  countDown2.startCountDown(60)
}



// 获取验证码按钮
const disabled = computed(() => countDown2.timeLeft.value > 0)

const buttonText = computed(() => disabled.value ? `${countDown2.timeLeft.value}秒后重试` : '获取验证码');

// el-tabs切换到选项卡‘账密登录’的时候，调用获取验证码的函数
const handleClick = (tab) => {
  if (tab.props.name === '2') {
    captchaStore.getIdentityCode()
  }

}

</script>

<template>
  <el-dialog v-model="showLoginDialog" width="35%" center :show-close="false">
    <el-tabs v-model="activeName" class="demo-tabs" :stretch="true" @tab-click="handleClick">
      <el-tab-pane label="免密登录" name="1">
        <el-form @keyup.enter="loginOrRegisterByCode(form1.email, form1.identifyCode1, activeName)"
                 ref="formRef1" :model="form1" :rules="rules1" label-width="70px" size="large" status-icon>
              <el-form-item prop="email" label="邮&nbsp;&nbsp;&nbsp;箱">
                <el-input v-model="form1.email" placeholder="请输入邮箱" clearable/>
              </el-form-item>
              <el-form-item prop="identifyCode1" label="验证码">
                <el-input v-model="form1.identifyCode1" placeholder="请输入验证码" clearable>
                    <template v-slot:append>
                        <el-button :disabled="disabled" @click="getIdentityCodeByEmail(form1.email)">{{buttonText}}</el-button>
                    </template>
                </el-input>
              </el-form-item>
              <small>未注册邮箱验证通过后将自动注册并登录</small>
              <el-form-item prop="agree" label-width="22px">
                <el-checkbox  size="large" v-model="form1.agree">
                  同意<a href="#">《隐私保护协议》</a> 和<a href="#">《服务条款》</a>
                </el-checkbox>
              </el-form-item>
              <el-button class="login" type="primary" @click="loginOrRegisterByCode(form1.email, form1.identifyCode1, activeName)">登录/注册</el-button>
        </el-form>
      </el-tab-pane>
      <el-tab-pane label="账密登录" name="2">
        <el-form @keyup.enter="loginOrRegisterByPwd(form2.email, form2.password, activeName,captchaStore.captchaInfo.captchaId, form2.captcha)"
                 ref="formRef2" :model="form2" :rules="rules2" label-width="70px" size="large" status-icon>
          <el-form-item prop="email" label="邮&nbsp;&nbsp;&nbsp;箱">
            <el-input v-model="form2.email" placeholder="请输入邮箱" clearable />
          </el-form-item>
          <el-form-item prop="password" label="密&nbsp;&nbsp;&nbsp;码">
            <el-input v-model="form2.password" type="password" show-password placeholder="请输入密码" clearable/>
          </el-form-item>
          <el-form-item prop="captcha" label="验证码">
            <el-input v-model="form2.captcha" placeholder="请输入验证码" clearable>
              <template v-slot:append>
                <img @click="captchaStore.getIdentityCode" :src="captchaStore.captchaInfo.picPath" style="height: 40px; width: 100px; cursor: pointer;" alt>
              </template>
            </el-input>
          </el-form-item>
          <small>未注册邮箱验证通过后将自动注册并登录</small>
          <el-form-item prop="agree" label-width="22px">
            <el-checkbox v-model="form2.agree" size="large">
              同意<a href="#">《隐私保护协议》</a> 和<a href="#">《服务条款》</a>
            </el-checkbox>
          </el-form-item>
          <el-button class="login2" @click="loginOrRegisterByPwd(form2.email, form2.password, activeName,captchaStore.captchaInfo.captchaId, form2.captcha)"
                     type="primary">登录/注册</el-button>
        </el-form>
      </el-tab-pane>
    </el-tabs>
  </el-dialog>
</template>

<style scoped>
.login, .login2 {
  width: 100%;
}
</style>
