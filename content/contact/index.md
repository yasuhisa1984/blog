---
title: "お問い合わせ"
date: 2024-12-20
draft: false
description: "お問い合わせはこちらのフォームからお願いします"
hidemeta: true
ShowToc: false
ShowBreadCrumbs: false
---

お仕事のご依頼・ご相談などは、下記フォームよりお気軽にお問い合わせください。

{{< rawhtml >}}
<form action="https://formspree.io/f/xnjnnpjg" method="POST" class="contact-form">
  <!-- ハニーポット（スパム対策）- ボットがこのフィールドに入力するとブロック -->
  <input type="text" name="_gotcha" style="display:none" tabindex="-1" autocomplete="off">

  <div class="form-group">
    <label for="name">お名前 <span class="required">*</span></label>
    <input type="text" id="name" name="name" required placeholder="山田 太郎">
  </div>

  <div class="form-group">
    <label for="email">メールアドレス <span class="required">*</span></label>
    <input type="email" id="email" name="email" required placeholder="example@example.com">
  </div>

  <div class="form-group">
    <label for="company">会社名（任意）</label>
    <input type="text" id="company" name="company" placeholder="株式会社〇〇">
  </div>

  <div class="form-group">
    <label for="subject">件名 <span class="required">*</span></label>
    <select id="subject" name="subject" required>
      <option value="">選択してください</option>
      <option value="お仕事のご依頼">お仕事のご依頼</option>
      <option value="技術相談">技術相談</option>
      <option value="その他のお問い合わせ">その他のお問い合わせ</option>
    </select>
  </div>

  <div class="form-group">
    <label for="message">メッセージ <span class="required">*</span></label>
    <textarea id="message" name="message" rows="6" required placeholder="お問い合わせ内容をご記入ください"></textarea>
  </div>

  <button type="submit" class="submit-btn">送信する</button>
</form>
{{< /rawhtml >}}

---

※ 通常2〜3営業日以内にご返信いたします。
