<!-- admin-config.vue —— 吃啥 · 系统配置（独立页，仅 H5/PC）
  从 admin.vue 的系统配置区整体拆出：系统级参数不再和业务数据（用户/技巧/菜谱/食材/餐厅）混在一起。
  入口：管理端左侧菜单「系统配置」。
  写入方式：input 用 v-model 绑本地 state，改完点底部「保存修改」统一落库（configs 表），
  只提交相对上次保存有改动的项；开关 / 分段选择这类点选即生效的控件仍即时保存。
-->
<template>
  <view class="cpage">
    <view class="cheader">
      <text class="back" @tap="back">‹</text>
      <text class="ctitle">系统配置</text>
      <text class="csub">配置表驱动 · 修改即生效</text>
    </view>

    <scroll-view class="cscroll" scroll-y>
      <view class="wrap">

        <!-- ============ 1. 基础配置 ============ -->
        <view class="gsec">
          <text class="g-h">基础配置</text>
          <view class="card">
            <view class="row">
              <view class="rc"><text class="rt">厨房技巧审核</text><text class="rs">新增/修改技巧需管理员审核</text></view>
              <switch :checked="audit" @change="onAudit" />
            </view>
            <view class="row">
              <view class="rc"><text class="rt">临期阈值（天）</text><text class="rs">在库食材剩余 ≤ 该天数视为临期</text></view>
              <input class="num w120" type="number" v-model="expiry" />
            </view>
            <view class="row">
              <view class="rc"><text class="rt">管理员口令</text><text class="rs">登录管理端使用，保存后立即生效</text></view>
              <input class="num w200" v-model="passcode" />
            </view>
          </view>
        </view>

        <!-- ============ 2. 图标池 ============ -->
        <view class="gsec">
          <text class="g-h">图标池</text>
          <text class="g-s">逗号分隔 emoji；录入表单的候选图标从这里取</text>
          <view class="card">
            <view class="col">
              <text class="rt">食材图标池</text>
              <textarea class="pool" v-model="ingPool" />
              <view class="prev"><text v-for="ic in ingIcons" :key="ic" class="p-ic">{{ ic }}</text></view>
            </view>
            <view class="col">
              <text class="rt">大类图标池</text>
              <textarea class="pool" v-model="catPool" />
              <view class="prev"><text v-for="ic in catIcons" :key="ic" class="p-ic">{{ ic }}</text></view>
            </view>
            <view class="col">
              <text class="rt">菜谱图标池</text>
              <textarea class="pool" v-model="recipePool" />
              <view class="prev"><text v-for="ic in recipeIcons" :key="ic" class="p-ic">{{ ic }}</text></view>
            </view>
            <view class="col">
              <text class="rt">餐厅图标池</text>
              <textarea class="pool" v-model="shopPool" />
              <view class="prev"><text v-for="ic in shopIcons" :key="ic" class="p-ic">{{ ic }}</text></view>
            </view>
            <view class="col last">
              <text class="rt">封面渐变池</text>
              <text class="rs">| 分隔 linear-gradient；菜谱封面预设</text>
              <textarea class="pool grad" v-model="gradPool" />
              <view class="prev"><view v-for="(g,i) in gradList" :key="i" class="p-grad" :style="{ background: g }"></view></view>
            </view>
          </view>
        </view>

        <!-- ============ 3. 邮件服务（SMTP） ============ -->
        <view class="gsec">
          <text class="g-h">邮件服务 · SMTP</text>
          <text class="g-s">用于发送邮箱验证码（登录 / 绑定 / 注销）。QQ 邮箱填「设置 → 账户 → 开启 SMTP 服务」生成的那串授权码，不是登录密码。</text>
          <view class="card">
            <view class="row">
              <view class="rc"><text class="rt">服务器 Host</text><text class="rs">QQ 邮箱：smtp.qq.com</text></view>
              <input class="num w220" v-model="smtpHost" placeholder="smtp.qq.com" />
            </view>
            <view class="row">
              <view class="rc"><text class="rt">端口 Port</text><text class="rs">465 = SSL，587 = STARTTLS</text></view>
              <input class="num w120" type="number" v-model="smtpPort" placeholder="465" />
            </view>
            <view class="row">
              <view class="rc"><text class="rt">发件邮箱 User</text><text class="rs">完整邮箱地址</text></view>
              <input class="num w260" v-model="smtpUser" placeholder="your_mail@qq.com" />
            </view>
            <view class="row">
              <view class="rc"><text class="rt">授权码 Pass</text><text class="rs">邮箱设置里生成的授权码（非登录密码）</text></view>
              <input class="num w260" type="password" v-model="smtpPass" placeholder="16 位授权码" />
            </view>
            <view class="row">
              <view class="rc"><text class="rt">发件人名称 From</text><text class="rs">收件人看到的发件人名</text></view>
              <input class="num w220" v-model="smtpFrom" placeholder="吃啥好呀" />
            </view>
            <view class="row">
              <view class="rc"><text class="rt">验证码有效期（分钟）</text><text class="rs">超时需重新获取</text></view>
              <input class="num w120" type="number" v-model="otpExpire" />
            </view>
            <view class="row">
              <view class="rc"><text class="rt">当前状态</text><text class="rs">{{ smtpReady ? '已配置，验证码会真实发信' : '未配置完整（缺 Host / User / Pass），验证码降级为后端日志打印' }}</text></view>
              <text class="badge" :class="{ on: smtpReady }">{{ smtpReady ? '已启用' : '未启用' }}</text>
            </view>
          </view>
        </view>

        <!-- ============ 4. 登录与安全 ============ -->
        <view class="gsec">
          <text class="g-h">登录与安全</text>
          <view class="card">
            <view class="col">
              <text class="rt">JWT 签名密钥</text>
              <text class="rs">用于签发登录 token，上线务必改成随机长串；改动后已发 token 全部失效</text>
              <input class="num full" v-model="jwtSecret" placeholder="随机长字符串" />
            </view>
            <view class="col">
              <text class="rt">微信小程序 AppID</text>
              <text class="rs">填好后小程序登录走真实 openid（不再 mock）</text>
              <input class="num full" v-model="mpAppid" placeholder="wx1234567890abcdef" />
            </view>
            <view class="col last">
              <text class="rt">微信小程序 AppSecret</text>
              <text class="rs">微信公众平台「开发 → 开发管理」里获取</text>
              <input class="num full" type="password" v-model="mpSecret" placeholder="AppSecret" />
            </view>
          </view>
        </view>

        <!-- ============ 5. 会员与收款 ============ -->
        <view class="gsec">
          <text class="g-h">会员与收款（VIP）</text>
          <text class="g-s">全部参数配置表驱动，改完即生效；额度/回看天数填 0 或留空 = 该项不限</text>
          <view class="card">
            <view class="row">
              <view class="rc"><text class="rt">会员体系开关</text><text class="rs">关 = 一切限制解除（额度/回看不生效）</text></view>
              <switch :checked="vipEnabled" @change="onVipEnabled" />
            </view>
            <view class="row">
              <view class="rc"><text class="rt">支付模式</text><text class="rs">mock=沙箱下单即开通（联调）；wxpay=微信虚拟支付（需签约+配商户参数）</text></view>
              <view class="modeseg">
                <text class="mopt" :class="{ on: vipMode === 'mock' }" @tap="setVipMode('mock')">mock</text>
                <text class="mopt" :class="{ on: vipMode === 'wxpay' }" @tap="setVipMode('wxpay')">wxpay</text>
              </view>
            </view>
            <view class="col">
              <text class="rt">免费额度（条）</text>
              <view class="fgrid">
                <view class="fitem"><text class="fl">菜谱</text><input class="num" type="number" v-model="fqRecipes" /></view>
                <view class="fitem"><text class="fl">食材</text><input class="num" type="number" v-model="fqIngredients" /></view>
                <view class="fitem"><text class="fl">餐厅</text><input class="num" type="number" v-model="fqShops" /></view>
                <view class="fitem"><text class="fl">技巧</text><input class="num" type="number" v-model="fqTips" /></view>
              </view>
            </view>
            <view class="col">
              <text class="rt">历史回看限制（天，仅非会员）</text>
              <view class="fgrid">
                <view class="fitem"><text class="fl">干饭记录</text><input class="num" type="number" v-model="fqRecordsDays" /></view>
                <view class="fitem"><text class="fl">体重</text><input class="num" type="number" v-model="fqWeightsDays" /></view>
              </view>
            </view>
            <view class="col">
              <text class="rt">套餐价格（元）</text>
              <view class="fgrid">
                <view class="fitem"><text class="fl">包月价</text><input class="num" type="digit" v-model="planMonthlyPrice" /></view>
                <view class="fitem"><text class="fl">包月天数</text><input class="num" type="number" v-model="planMonthlyDays" /></view>
                <view class="fitem"><text class="fl">年卡价</text><input class="num" type="digit" v-model="planYearlyPrice" /></view>
                <view class="fitem"><text class="fl">年卡天数</text><input class="num" type="number" v-model="planYearlyDays" /></view>
              </view>
            </view>
            <view class="col">
              <text class="rt">虚拟支付 Offer ID</text>
              <text class="rs">小程序后台「虚拟支付 - 基础配置」里获取</text>
              <input class="num full" v-model="wxpayOfferId" placeholder="wx开头的offerId" />
            </view>
            <view class="col last">
              <text class="rt">虚拟支付 Pay Key</text>
              <text class="rs">用于支付签名与查单对账，务必保密</text>
              <input class="num full" type="password" v-model="wxpayPayKey" placeholder="pay_key" />
            </view>
          </view>
        </view>

        <view class="tail">改完点下方「保存修改」统一生效；开关与支付模式点选即保存</view>
      </view>
    </scroll-view>

    <!-- 底部保存栏：始终可见，无改动时置灰 -->
    <view class="cfoot">
      <text class="cf-hint">{{ dirtyCount ? '有 ' + dirtyCount + ' 项修改未保存' : '暂无修改' }}</text>
      <button class="cf-btn" :class="{ dis: !dirtyCount || saving }" :disabled="!dirtyCount || saving" @tap="saveAll">
        {{ saving ? '保存中…' : '保存修改' }}
      </button>
    </view>
  </view>
</template>

<script>
import { configApi, adminApi } from '@/api'
import request from '@/utils/request'

export default {
  data() {
    return {
      // 门控：admin-config 是管理台专属页，必须持有有效 admin token 才能读/写系统配置。
      // 没登管理台直接进来时，storage 里既无 eat_admin 也无 eat_admin_token：
      //   GET 非敏感键 200、敏感键（ADMIN_ONLY_KEYS）403；
      //   PUT 任何键一律 403（写接口用 get_admin_user）——保存会直接失败。
      // 所以 onLoad 先用 adminApi.stats() 验一次管理员身份，不过就跳回 admin.vue 让用户先登。
      loading: true,          // 读配置中（此时不算「有改动」）
      saving: false,          // 保存请求进行中
      snap: {},               // 上次已落库的值快照，用于算改动项
      audit: true, expiry: '3', passcode: '',
      ingPool: '', catPool: '', recipePool: '', shopPool: '', gradPool: '',
      smtpHost: '', smtpPort: '465', smtpUser: '', smtpPass: '', smtpFrom: '', otpExpire: '10',
      jwtSecret: '', mpAppid: '', mpSecret: '',
      // 会员与收款
      vipEnabled: false, vipMode: 'mock',
      fqRecipes: '20', fqIngredients: '50', fqShops: '10', fqTips: '10',
      fqRecordsDays: '60', fqWeightsDays: '90',
      planMonthlyPrice: '6', planMonthlyDays: '30', planYearlyPrice: '45', planYearlyDays: '365',
      wxpayOfferId: '', wxpayPayKey: '',
    }
  },
  computed: {
    gradList() {
      return (this.gradPool || '').split('|').map((s) => s.trim()).filter(Boolean)
    },
    smtpReady() {
      return !!(this.smtpHost && this.smtpUser && this.smtpPass)
    },
    // 图标预览：直接由池文本派生，输入即时更新（无需等保存）
    ingIcons() { return this.splitIcons(this.ingPool) },
    catIcons() { return this.splitIcons(this.catPool) },
    recipeIcons() { return this.splitIcons(this.recipePool) },
    shopIcons() { return this.splitIcons(this.shopPool) },
    // 表单当前值 → { configKey: 字符串 }，与后端 configs 表 key 一一对应。
    // free_quota / vip_plans 按固定键序序列化，保证与快照比对时字符串稳定。
    formValues() {
      const t = (v) => String(v == null ? '' : v).trim()
      return {
        expiry_threshold_days: t(this.expiry),
        admin_passcode: t(this.passcode),
        ingredient_icon_pool: t(this.ingPool),
        cat_icon_pool: t(this.catPool),
        recipe_emoji_pool: t(this.recipePool),
        shop_icon_pool: t(this.shopPool),
        cover_grad_pool: t(this.gradPool),
        smtp_host: t(this.smtpHost),
        smtp_port: t(this.smtpPort),
        smtp_user: t(this.smtpUser),
        smtp_pass: t(this.smtpPass),
        smtp_from: t(this.smtpFrom),
        otp_expire_min: t(this.otpExpire),
        jwt_secret: t(this.jwtSecret),
        mp_appid: t(this.mpAppid),
        mp_secret: t(this.mpSecret),
        free_quota: JSON.stringify({
          recipes: Number(this.fqRecipes) || 0,
          ingredients: Number(this.fqIngredients) || 0,
          shops: Number(this.fqShops) || 0,
          tips: Number(this.fqTips) || 0,
          records_view_days: Number(this.fqRecordsDays) || 0,
          weights_view_days: Number(this.fqWeightsDays) || 0,
        }),
        vip_plans: JSON.stringify(this.buildPlans()),
        wxpay_offer_id: t(this.wxpayOfferId),
        wxpay_pay_key: t(this.wxpayPayKey),
      }
    },
    // 相对快照有改动的 key（读取中不算）
    dirtyKeys() {
      if (this.loading) return []
      const cur = this.formValues
      return Object.keys(cur).filter((k) => cur[k] !== this.snap[k])
    },
    dirtyCount() { return this.dirtyKeys.length },
  },
  onLoad() {
    // 管理台专属页：必须持有有效 admin token 才能继续。
    // 本地标记可伪造，统一以后端校验为准（stats 仅管理员可读），失败弹回登录门。
    const hasAdmin = uni.getStorageSync('eat_admin') === '1'
    const hasToken = !!uni.getStorageSync(request.ADMIN_TOKEN_KEY)
    if (!hasAdmin || !hasToken) {
      this._kickBack()
      return
    }
    adminApi.stats()
      .then(() => this.load())
      .catch(() => this._kickBack())
  },
  methods: {
    _kickBack() {
      uni.removeStorageSync('eat_admin')
      uni.removeStorageSync(request.ADMIN_TOKEN_KEY)
      uni.showToast({ title: '请先在管理台登录', icon: 'none' })
      setTimeout(() => uni.reLaunch({ url: '/pages/admin/admin' }), 600)
    },

    back() {
      // 有未保存改动时确认离开，避免误触丢改动
      if (this.dirtyCount) {
        uni.showModal({
          title: '放弃未保存的修改？',
          content: `有 ${this.dirtyCount} 项修改尚未保存，离开将丢失。`,
          confirmText: '放弃离开', confirmColor: '#e64340',
          success: (res) => { if (res.confirm) uni.navigateBack() },
        })
        return
      }
      uni.navigateBack()
    },

    splitIcons(v) {
      return (v || '').split(/[,，]/).map((s) => s.trim()).filter(Boolean)
    },

    // 套餐对象：价格按元输入，落库转分（与历史上保存格式保持一致）
    buildPlans() {
      const yuan2fen = (v) => Math.round(Number(v) * 100)
      return {
        monthly: { name: '连续包月', price_fen: yuan2fen(this.planMonthlyPrice), days: Number(this.planMonthlyDays) || 0 },
        yearly: { name: '年卡', price_fen: yuan2fen(this.planYearlyPrice), days: Number(this.planYearlyDays) || 0, hot: 1 },
      }
    },

    async load() {
      try {
        const keys = [
          'audit_enabled', 'expiry_threshold_days', 'admin_passcode',
          'ingredient_icon_pool', 'cat_icon_pool', 'recipe_emoji_pool', 'shop_icon_pool', 'cover_grad_pool',
          'smtp_host', 'smtp_port', 'smtp_user', 'smtp_pass', 'smtp_from', 'otp_expire_min',
          'jwt_secret', 'mp_appid', 'mp_secret',
          'vip_enabled', 'vip_pay_mode', 'free_quota', 'vip_plans', 'wxpay_offer_id', 'wxpay_pay_key',
        ]
        const res = await Promise.all(keys.map((k) => configApi.get(k).catch(() => ({ value: '' }))))
        const v = {}
        keys.forEach((k, i) => { v[k] = (res[i] && res[i].value != null) ? String(res[i].value) : '' })

        this.audit = v.audit_enabled === '1' || v.audit_enabled === 'true'
        this.expiry = v.expiry_threshold_days || '3'
        this.passcode = v.admin_passcode
        this.ingPool = v.ingredient_icon_pool
        this.catPool = v.cat_icon_pool
        this.recipePool = v.recipe_emoji_pool
        this.shopPool = v.shop_icon_pool
        this.gradPool = v.cover_grad_pool
        this.smtpHost = v.smtp_host
        this.smtpPort = v.smtp_port || '465'
        this.smtpUser = v.smtp_user
        this.smtpPass = v.smtp_pass
        this.smtpFrom = v.smtp_from
        this.otpExpire = v.otp_expire_min || '10'
        this.jwtSecret = v.jwt_secret
        this.mpAppid = v.mp_appid
        this.mpSecret = v.mp_secret
        // 会员与收款（free_quota / vip_plans 是 JSON，解析失败回默认）
        this.vipEnabled = v.vip_enabled === '1'
        this.vipMode = v.vip_pay_mode || 'mock'
        try {
          const fq = JSON.parse(v.free_quota || '{}')
          this.fqRecipes = String(fq.recipes ?? 20)
          this.fqIngredients = String(fq.ingredients ?? 50)
          this.fqShops = String(fq.shops ?? 10)
          this.fqTips = String(fq.tips ?? 10)
          this.fqRecordsDays = String(fq.records_view_days ?? 60)
          this.fqWeightsDays = String(fq.weights_view_days ?? 90)
        } catch (e) {}
        try {
          const vp = JSON.parse(v.vip_plans || '{}')
          this.planMonthlyPrice = String((vp.monthly?.price_fen ?? 600) / 100)
          this.planMonthlyDays = String(vp.monthly?.days ?? 30)
          this.planYearlyPrice = String((vp.yearly?.price_fen ?? 4500) / 100)
          this.planYearlyDays = String(vp.yearly?.days ?? 365)
        } catch (e) {}
        this.wxpayOfferId = v.wxpay_offer_id
        this.wxpayPayKey = v.wxpay_pay_key
      } catch (e) {
        uni.showToast({ title: e.message || '加载失败', icon: 'none' })
      } finally {
        // 落库基准：读完把当前值存为快照，此后只有再改才算「有改动」
        this.snap = this.formValues
        this.loading = false
      }
    },

    // 统一保存：只提交有改动的 key，全部成功后刷新快照
    async saveAll() {
      const keys = this.dirtyKeys
      if (!keys.length || this.saving) return

      const posInt = (v) => { const n = Number(v); return Number.isFinite(n) && n > 0 }
      if (keys.indexOf('expiry_threshold_days') >= 0 && !posInt(this.expiry)) {
        return uni.showToast({ title: '临期阈值需为正整数', icon: 'none' })
      }
      if (keys.indexOf('otp_expire_min') >= 0 && !posInt(this.otpExpire)) {
        return uni.showToast({ title: '验证码有效期需为正整数', icon: 'none' })
      }
      if (keys.indexOf('vip_plans') >= 0) {
        const p = this.buildPlans()
        if (p.monthly.price_fen <= 0 || p.yearly.price_fen <= 0) {
          return uni.showToast({ title: '价格需大于 0 元', icon: 'none' })
        }
      }

      this.saving = true
      const vals = this.formValues
      // jwt_secret 一改，已签发的 token（含当前管理台令牌）当场失效，
      // 所以放到最后写：其余配置先落库，不会因它中途失效而漏写。
      const ordered = keys.filter((k) => k !== 'jwt_secret')
        .concat(keys.filter((k) => k === 'jwt_secret'))
      try {
        // 串行提交，任一失败即中断；失败不刷新快照，未写成功的项仍标记为「有改动」
        for (let i = 0; i < ordered.length; i++) {
          await configApi.set(ordered[i], vals[ordered[i]])
        }
        this.snap = vals
        uni.showToast({ title: `已保存 ${ordered.length} 项`, icon: 'success', duration: 900 })
      } catch (e) {
        // 只有「请先登录后再操作」（401，令牌失效/未登录）才清会话跳登录；
        // 「需要管理员权限」（403）是角色问题，令牌还有效，不该清会话。
        const msg = e.message || ''
        if (msg.indexOf('请先登录') >= 0) {
          uni.removeStorageSync('eat_admin')
          uni.removeStorageSync(request.ADMIN_TOKEN_KEY)
          uni.showToast({ title: '登录已过期，请重新登录', icon: 'none' })
          setTimeout(() => uni.reLaunch({ url: '/pages/admin/admin' }), 800)
        } else {
          uni.showToast({ title: '保存失败：' + msg, icon: 'none' })
        }
      } finally {
        this.saving = false
      }
    },

    // ===== 点选即生效的控件（开关 / 分段）=====
    onAudit(e) {
      this.audit = e.detail.value
      configApi.set('audit_enabled', this.audit ? '1' : '0')
        .then(() => uni.showToast({ title: '已保存', icon: 'none', duration: 700 }))
        .catch((err) => uni.showToast({ title: err.message || '保存失败', icon: 'none' }))
    },
    onVipEnabled(e) {
      this.vipEnabled = e.detail.value
      configApi.set('vip_enabled', this.vipEnabled ? '1' : '0')
        .then(() => uni.showToast({ title: '已保存', icon: 'none', duration: 700 }))
        .catch((err) => uni.showToast({ title: err.message || '保存失败', icon: 'none' }))
    },
    setVipMode(m) {
      this.vipMode = m
      configApi.set('vip_pay_mode', m)
        .then(() => uni.showToast({ title: '已保存', icon: 'none', duration: 700 }))
        .catch((err) => uni.showToast({ title: err.message || '保存失败', icon: 'none' }))
    },
  },
}
</script>

<style lang="scss" scoped>
.cpage { display:flex; flex-direction:column; height:100vh; background:var(--bg); }
.cheader { display:flex; align-items:baseline; gap:16rpx; padding:calc(env(safe-area-inset-top) + 20rpx) var(--nav-safe-right) 20rpx 24rpx; }
.back { font-size:48rpx; font-weight:600; }
.ctitle { font-size:40rpx; font-weight:700; }
.csub { font-size:24rpx; color:var(--text-2); }
.cscroll { flex:1; min-height:0; }

.wrap { padding:0 24rpx; max-width:1100px; margin:0 auto; }
.gsec { margin-bottom:28rpx; }
.g-h { font-size:30rpx; font-weight:700; display:block; }
.g-s { font-size:22rpx; color:var(--text-2); display:block; margin:6rpx 0 12rpx; line-height:1.6; }
.card { background:var(--card); border:1rpx solid var(--border); border-radius:16rpx; padding:8rpx 20rpx; }

.row { display:flex; align-items:center; gap:20rpx; padding:18rpx 0; }
.row + .row { border-top:1rpx solid var(--border); }
.rc { flex:1; min-width:0; display:flex; flex-direction:column; gap:4rpx; }
.rt { font-size:26rpx; font-weight:600; }
.rs { font-size:22rpx; color:var(--text-2); line-height:1.5; }

.col { padding:18rpx 0; display:flex; flex-direction:column; gap:8rpx; }
.col + .col { border-top:1rpx solid var(--border); }
.col.last { padding-bottom:22rpx; }

.num { background:var(--bg); border:1rpx solid var(--border); border-radius:10rpx; height:70rpx; line-height:70rpx; padding:0 16rpx; font-size:26rpx; color:var(--text); box-sizing:border-box; flex:none; }
.num.full { width:100%; flex:1 1 auto; }
.w120 { width:120px; } .w220 { width:220px; } .w260 { width:260px; }

.pool { background:var(--bg); border:1rpx solid var(--border); border-radius:10rpx; width:100%; box-sizing:border-box; min-height:110rpx; padding:12rpx 16rpx; font-size:24rpx; color:var(--text); }
.pool.grad { min-height:150rpx; }
.prev { display:flex; flex-wrap:wrap; gap:8rpx; margin-top:6rpx; }
.p-ic { background:var(--bg); border-radius:8rpx; padding:4rpx 10rpx; font-size:26rpx; }
.p-grad { width:56rpx; height:34rpx; border-radius:8rpx; }

.badge { font-size:22rpx; padding:6rpx 16rpx; border-radius:999rpx; background:#fdecea; color:#c0392b; flex:none; }
.badge.on { background:#e8f8ef; color:#0a8f4d; }

.tail { text-align:center; font-size:22rpx; color:var(--text-2); padding:8rpx 0 20rpx; }
/* 会员与收款 */
.modeseg { display:flex; border:1rpx solid var(--border,#e5e5e5); border-radius:12rpx; overflow:hidden; }
.mopt { padding:10rpx 28rpx; font-size:26rpx; color:var(--text-2); background:#f7f7f9; }
.mopt.on { color:#fff; background:var(--brand,#4b3fe3); font-weight:700; }
.fgrid { display:flex; flex-wrap:wrap; gap:16rpx; margin-top:10rpx; }
.fitem { display:flex; align-items:center; gap:10rpx; }
.fl { font-size:24rpx; color:var(--text-2); }
.fitem .num { width:140rpx; }

/* 底部保存栏 */
.cfoot { display:flex; align-items:center; gap:20rpx; padding:16rpx 24rpx calc(env(safe-area-inset-bottom) + 16rpx); background:var(--card); border-top:1rpx solid var(--border); }
.cf-hint { flex:1; font-size:24rpx; color:var(--text-2); }
.cf-btn { height:76rpx; line-height:76rpx; padding:0 48rpx; border-radius:12rpx; background:var(--brand); color:#fff; font-size:28rpx; font-weight:600; border:none; margin:0; }
.cf-btn::after { border:none; }
.cf-btn.dis { background:#c9c9d2; }
.cf-btn[disabled] { color:#fff; background:#c9c9d2; }
</style>
