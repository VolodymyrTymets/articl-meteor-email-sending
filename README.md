# Організація відправки емейлів у meteor js

Відправка емайлів досить поширена задача, яка виникає у багатьох проеках. Відправка одного чи двох повідомлень не повинна викликати будь яких проблем. Більшість фрейморків надають безліч засобів для вирішення цієї задачі. І метеор не виключення. Він надає легкий і зрозумілий [інтерфейс](https://docs.meteor.com/api/email.html) для відправки повідомлень.  Але, що якщо потрбно організувати відправку багатьох емейлів, з різними шаблонами, футерами та у відповідності до різних подій. А також внедрити цю систему у великий робочий додаток. У цій статті наведено один з можливих варіантів вирішення ціїє проблеми.
### Проблеми які слід вирішити 
При розробці складної системи відправки емелів я стикнувся із рядом проблем. Вирішиви їх можна значно скороти час розробки, полекшити розуміння системи а отже і супровід в майбутньому. Нижче наведено ряд проблем які слід вирішити:   
- побудова гнучких диначічних шаблонів;
- динамічна побудова footers для кожного шаблону;
- імплементація відправки майлів через увесь додаток.
- побудова ateches files для їх відправки у мейлах;

## Робота із шаблонами листів 
Дійсно складні листи, призначенні для вирішення реальних бізнес завдань можуть містити дісно велику кількість інформації та  стилів. При цьому не рідко приходиться вставляти ці ж стилі у header листа. Прицьому стилі можуть бути різними обо однаковими для різних листів. Наприклад лист запрошення користувача та  підтвердження емайлу можуть мати однаковий шаблон, а лист звіту по продажам за місяць зовсім іншу. Теж стосується і foters. Інформація (сам текст листа не завжди є статичним) наприклад той же самий лист звіту по продажам може генеруватися раз в місяць і використовувати різні дані додатку. Припустимо замовнику слід відправити кілька листів з наступним виглядом:

![](https://image.ibb.co/mkkfrQ/email_1.png)

При чому, як уже було сказано раніше наповнення таких листів може мінятися відповідно до специфіки завдання та генеруватися динамічно, використовуючи різні дані системи. Було б логічно розділити даний лист на наступні складові.

![](https://image.ibb.co/hGdN5k/email_2.png)

* [Layout] - основний макет листа. Зазвичай усі листи матимуть приблизно одинаковий набір стилів заголовків та інхиш нюансів. Вони з малою імовірністю будуть змінюватися. Тому їх варто винести окремо що не повторювати самих себе і не дублювати код, навіть якщо це код розмітки. 
* [Body of email] - основний текст листа. Цей контент буде різний для кожного листа.
* [Footer] - даний контент може бути як різним так і однаковим для різних листів.

Перейдем до практичної реалізації розглянутого вище рішення. Перш за все слід створити Laout. Для цього створимо layout.html файл знастуною розміткою.
```
<html xmlns="http://www.w3.org/1999/xhtml">
<head>
  <meta http-equiv="Content-Type" content="text/html; charset=UTF-8" />
  <title>{{subject}}</title> <!-- Title of email -->
  <style ...></style> <!-- style of email if need -->
</head>
<body itemscope itemtype="http://schema.org/EmailMessage">
    {{{email}}}  <!-- body of email -->
    <footer>
      {{{footer}}} <!-- footer of email -->
    </footer>
</body>
</html>
```
> Зверніть увагу що використона `{{{email}}}` замість `{{email}}` це означає що праметер `email` може бути повноцінною html розміткою. Детальніше про реалізацію відправки розказано нижче. 

Наступним кроком слід створити footer. Оскільки їх може бути декілька назвем його footerA.html. Для конкретних завдань  можна називати як потрібно.
```
<div>
  Hello im footer A from {{nameOfApp}}
<div>
```
Footer може містити довільну розмітку із динамічними даними. В даному випадку `nameOfApp` буде імя додатку. Тепер залишилося тільки створити сам лист. Назвем його `invitationEmail.html`.
```
Hello {{usernName}} you invitated for {{nameOfApp}}. To confirm please cleck here: {{invitationUrl}}
```
### Побудова та відправка шаблонів
Перед тим як відправити шаблони повідомлень спершу їх слід відрендерити. Для цього можна використати пакет [SSR](https://atmospherejs.com/meteorhacks/ssr). Також знадобитьтся Assets клас щоб прочитати відповідний html файл. Загалом код виглядатиме наступним.
```
SSR.compileTemplate(templateName, Assets.getText(`${templateName}.html`));
cont html = SSR.render(templateName, { ...data });
  Email.send({
        to: recipient,
        from: `FindEd ${email}`,
        subject: subject,
        html: html,
      });
```

Проте прописувати такий код в кожному місці нашого додатку не дуже хороше рішення. До того ж не зрозуміло до чого тут наші `footer` та `layout`. Тому наступним кроком буде побудова `EmailSender` клас. Його використання матиме наступний вигляд.
```
  new EmailSender('Subject of email', 'invitationEmail', { username, nameOfApp }).
       withFooter('footerA', { nameOfApp }).withAttachments([attachments]).sendFor(['email1', 'email2']);
```
Погодтеся такий клас виглядає значно елегантніше і більш зрозумілий у використанні. Методи `withFooter` і `withAttachments` можуть бути пропущенні якщо емейлу не потрібний `footer` чи `withAttachments`. Перейдем беспосереднбо до реалізації даного клсу.

## Реалізаці класу EmailSender
Спершу глянемо на сруктуру класу і поговоримо про методи які нам потрубно реалізувати.
```
class EmailSender {
  constructor(subject, templateName, templateDate) {
  }

  withAttachments(attachments) { }
  withFooter(footerName, data = {}) { }
  _renderTemplate() { }
  _sendEmail(recipient) { }
  sendFor(recipients) { }
}
```

* [sendFor] - виклик даного методу ініцієює відправку повідомлення для користувачів;
* [_sendEmail] - відправляє повідомлення для одного користувача;
* [_renderTemplate] - відповідає за побудому шаблону повідомлення;
* [withFooter] - приймає і зберігає дані для footer які використовуються для побудови шаблону.  
* [withAttachments] - приймає Attachments (масив pdf файлів) і зберігає для подальшої відправки;


Тепер можна переходити до безпосередньої реалізації кожного методу. 
#### EmailSender constructor
```
 /**
    @param <String> - subject of mesage
    @param <String> - name of template to send
    @param <Object> - date of tempkate
 **/
 constructor(subject, templateName, templateDate) {
    this._senderEmail = Meteor.settings.from; // sender email
    this._emailTemplatePath = 'templates/emails/'; // path for folder with email templates
    this._footerTemplatePath = `${this._emailTemplatePath}footers/`; // path for folder with footer templates
    this._layotName = 'layout';

    this._subject = subject;
    this._templateName = templateName;
    this._templateDate = templateDate;
    this._template = null;
    this._footer = null;

    this._sendEmail = this._sendEmail.bind(this);
  }
```
Констуктор ініціалізує дані для роботи класу. Спершу потрібно зберегти емейл відправника `_senderEmail` його можна брати з начтройок або прописувати прямо в конструкторі якщо це не критично. Далі слід  сберегти шлях до шаблонів. У метеор додатку такі файли зберігаються у папці `private`. Тому розмістіть ваші темплати та 'layout' у папці `private/templates/emails` а footer темплейти у `private/templates/footers`. Ці шляхи можуть бути довільні, але не забуть вказати їх у кострукторі.  
#### EmailSender sendFor
```
/**
  @param [Object] email addresses
**/
  sendFor(recipients) {
    this.renderTemplate();
    const emails = _.isArray(recipients) ? recipients : [recipients];
    emails.forEach(this._sendEmail);
  }
```
Досить простий метод який викликає метод для рендерингу шаблонів та відправки повідомення.
#### EmailSender _sendMail
```
/**
  @private
  @param <String> - mail of recipient
**/
  _sendEmail(recipient) {
    Meteor.defer(() => {
      Email.send({
        to: recipient,
        from: this._senderEmail,
        subject: this._subject,
        html: this._template,
        attachments: this._attachments,
      });
    });
  }
```
Диний метод викликає метеорівськи  метод  Email.send для відправки повідомлень.
> [  Meteor.defer](https://docs.meteor.com/api/core.html#Meteor-defer) використано для того щоб відправка повідомлень відбувалася у baground mode. Щоб не очікувати поки повідомлення відправляться. Також опціонально для кожного додатку.
#### EmailSender _renderTemplate
```
  // @private 
  _renderTemplate() {
    SSR.compileTemplate(this._templateName, // compile email template
        Assets.getText(`${this._emailTemplatePath}${this._templateName}.html`));
    SSR.compileTemplate(this._layotName, // compile layout template
        Assets.getText(`${this._emailTemplatePath}${this._layotName}.html`));

   const emailTemplate = SSR.render(this._templateName, this._templateDate);
   this._template =  SSR.render(this._layotName, { email: emailTemplate,  subject: this._subject, footer: this._footer });
  }
```
Даний метод відповідальний за рендерінг шаблону повідомлення. Спочатку редеряться шаблон повідомлення та footer. Після цбого рендерится сам layout якому передані шаблон повідомлення та footer як параметри. Далі це html зберігається для відправки у листі.
#### EmailSender withFooter
```
/**
  @param <String> - name of footer
  @param <Object> - data of footer
**/
  withFooter(footerName, data = {}) {
    SSR.compileTemplate(footerName, // compile footer
      Assets.getText(`${this._footerTemplatePath}${footerName}.html`));
    this._footer = SSR.render(footerName, { rootUrl, logoUrl, siteName, ...data });
    return this;
  }
```
Даний метод редерить шаблон footer і зберігає його для подальшого відправлення у листі.
> Важливо в кінці метода викликати `rerutn this` щоб можна було організувати цепочку викликів.
#### EmailSender withFooter
```
/**
  @param <Object> array of attachments pdf file for example
**/
  withAttachments(attachments) {
    this._attachments = attachments;
    return this;
  }
```
даний метод зберігаї attachments для подальшої відправки у листі. Що це за attachments і як їх реалізувати розказано нижче.

Отже такий невеличкий клас дозволить вам будувати гнучкі обширні шаблони повідомлення уникаючи великого дублювання коду. Як і у шаблонах так і у місцях відправки повідомлень.

## Робота з Attachments

Не рідко виникає потреба відправити у листі певний файл звітності чи щось у цьому роді. Розглянемо спосіб побудови та відправки pdf файлів у meteor додатку. Для побудови pdf файлу можна використати такіж шаблони як ми використовували вище. А для перетворення html у pdf можна використати пакет [html-pdf](https://atmospherejs.com/jgdovin/html-pdf). 
Спершу створимо файли `invitationDocument.thml` і розмістимо його у дерикторії `private/templates/pdfs` нашого проекту.
```
<html xmlns="http://www.w3.org/1999/xhtml">
<head>
  <meta http-equiv="Content-Type" content="text/html; charset=UTF-8" />
  <title>{{subject}}</title> <!-- Title of email -->
  <style ...></style> <!-- style of email if need -->
</head>
<body itemscope itemtype="http://schema.org/EmailMessage">
   Welcome into {{nameOfApp}}
</body>
</html>
```
> Для простоти даний документ містить усю розмітку. Але яко необхідно використовувати макети ви можете це робити як у випадку з емейлами.

Наступним кроком буде побудова `PdfBuilder`. В результаті якого ми отримає досить зручний код викилику побудови pdf файлів:
```
new PdfBuilder('template_name', {...data }).build('pdfFileName', pdf => ...)   
```
#### Реалізаці класу PdfBuilder
```
/**
  @param <String> - name of template for rendering 
  @param <Object> - data for template
**/
class PdfBuilder {
  constructor(templateName, templateDate) {
    this._templatePath = 'templates/pdfs/';
    this._templateName = templateName;
    this._templateDate = templateDate;
  }
  _renderTemplate() {
    SSR.compileTemplate(this._templateName,
      Assets.getText(`${this._templatePath}${this._templateName}.html`));
    return SSR.render(this._templateName, this._templateDate);
  }
  build(fileName, ...callbacks) {
    pdf.create(this._renderTemplate(),  { format: 'Letter' }).toStream(
      Meteor.bindEnvironment((error, stream) => {
        if (!error) {
          const attachedPDF = { fileName, streamSource: stream};
          callbacks.forEach(callback => callback(attachedPDF));
        } else ...
      }));
  }
}
```

Досить простий клас який ініціалізує дані для pdf докуметна у конструкторі. Редерить сам шаблон документа у методі `_renderTemplate`. Останнім кроком є створення власне pdf файлу у методі `build` та передає його у calback.
> Нам потрібен calback оскільки машині потрібний певний час для створення цього фалу. І власне така реалізація пакету. 

## Підсумок
Ми розділили на уявний лист на кілька шаблонів щоб уникнути дублювання великої розмітнки у кожному новому листі. Створили та організували наші шаблони. Написали наш клас, який допомогає легко організувати відправку листів у великій системі. А також організували можливість прикріпляти файли до цих же листів. Отже, тепер для того щоб відправити лист запрошення користувачам нашого додатку із прикріпленим pdf фалом, достатньо викликати:
```
const nameOfApp = 'Awesome App';
new PdfBuilder('invitationDocument', { nameOfApp }).build('pdfFileName', pdf => 
 new EmailSender('Subject of email', 'invitationEmail', { username, nameOfApp }).
       withFooter('footerA', { nameOfApp }).withAttachments([pdf]).sendFor(['email1@gmail.com', 'email2@gmail.com'])); 
```
Всього три стрічки які можна помісти у будь якому місці вашого проекту.

## Остання порада
Не рідко виникає необхідність вказувати у листах посилання на сайт розроблюваної системи. Проблема в тому що система може міняти свій шлях. Або ж можуть бути два сервери один для розробки один для роботи додатку. Тому просто прописувати шлях у шаблонах не найкращий варіант. Потрібно мати механізм для того щоб генерувати root url відповідно того де хоститься додаток. Наведений нижче метод дозволяє згенерувати потрібний шлях разом із параметрами.
```
 static hrefFor(path, data) {
    const { ROOT_URL } = process.env;
    const domainPath = ROOT_URL.slice(-1) === '/' && ROOT_URL || `${ROOT_URL}/`;
    const dataToPath = key => `${data[key]}/`;
    const datePath = data ? Object.keys(data).map(dataToPath).join('') : '';

    return `${domainPath}${path}${datePath}`;
  }
```
> Даний метод є частиною `EmailSender` і являється статичним для зручного використання.

Отже якщо необхіжно згенерувати повний шлях до сторінки `user-list` достатньо викликати `EmaisSender.hrefFor('/user-list')` і отриманий шлях буде `https://www.google.com.ua/user-list`. 

