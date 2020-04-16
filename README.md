### Tóm tắt
##### 1 [Một số ý tưởng về thiết kế hệ thống và quy trình người dùng ](#user)
###### 1.1 [Quản lý user và role](#user-role)
###### 1.2 [Khóa tài khoản khi sai password](#blockage)
###### 1.3 [Shortcut](#shortcut)
###### 1.1 [Hệ thống field validation](#validation)

##### 2 [Một số kỹ thuật tham khảo](#techniques)  
###### 2.1 [ValidateWrapper](#validate-wrapper)
###### 2.2 [BaseButton](#basebutton)
###### 2.3 [Field Validation](#field-validation)

### Nội dung chính

<a name="user"/>

### 1 Một số ý tưởng về thiết kế hệ thống và quy trình người dùng 

<a name="user-role"/>

##### 1.1 Quản lý user và role
Có 3 bảng chính là `User`, `ActionPoint` và `Role` và 2 bảng giữa là `RoleAction` và `UserRole`
- Bảng `User` lưu thông tin user
- Bảng `ActionPoint` lưu thông tin của một hành động, VD: in, gửi, xóa ...
- Bảng `Role` lưu thông tin của một role, VD: bác sĩ, y tá, admin, ...
- Bảng `RoleAction` lưu Role A được làm hành động B và khi thực hiện A có cần confirm (nhập password) hay không (một role có thể làm nhiều hành động)
- Bảng `UserRole` lưu User A có role là B (một user có thể có nhiều role)

<a name="blockage"/>

##### 1.2 Khóa tài khoản khi sai password
Khi user đăng nhập một tài khoản đang tồn tại (existing user) mà sai password N lần (N được hard-coded sẵn hoặc lưu database thì tùy thiết kế) thì sẽ bị khóa M phút

<a name="shortcut"/>

##### 1.3 Shortcut
Mỗi hành động `ActionPoint` hầu hết đều được đại diện bởi một button ở front-end, và button này cần có một short cut, VD: nút lưu cần có short cut là Ctrl-S
Cụ thể kỹ thuật xem 2.3

<a name="validation"/>

##### 1.4 Hệ thống field validation
Ví dụ: khi có UI thêm một bệnh nhân vào database, các field cần được validate dựa vào một số điều kiện

Parameter | Type | Min length | Max length | Min value | Max value | Required
:--- | :--- | :--- | :--- | :---  | :--- | :---
PatientName | Text | 6 | 25 | N/A | N/A | Yes
PatientAge | Number | N/A | N/A | 1 | N/A | No

Mỗi loại dữ liệu (VD Patient, User, Transaction, ...) đều có một "bảng" validation như trên,

##### 

<a name="techniques"/>

### 2. Một số kỹ thuật tham khảo

<a name="validate-wrapper"/>

##### 2.1 ValidateWrapper
Mỗi hành động đều được gắn với một `ActionPoint`, trước khi hành động được thực hiện thì cần bắn API kiểm tra xem user đó có quyền thực hiện hành động đó hay không hoặc nếu được thực hiện thì có cần confirm (nhập password) hay không
Ý tưởng của `ValidateWrapper` là tiền xử lý props ở constructor, lấy các props có type là function ra
```jsx
class ValidateWrapper extends BaseConsumer {
    constructor(props) {
        super(props);
        this._otherProps = {};
        this._eventFunction = {};
        Object.keys(this.props).forEach(prop => {
            if (typeof this.props[prop] === "function") {
                this._eventFunction[prop] = (...params) => {
                    let action = () => {
                        this.props[prop](...params);
                    }
                    this._validate(action);
                }
            } else {
                this._otherProps[props] = this.props[prop];
            }
        });
    }
    _validate = action => {
        // validate ...
    }
    consumerContent() {
        let { children, actionId, ...others } = this.props;
        // tại đây nên có option rằng nếu user không có quyền thực hiện hành động có có display cái nút hay không
        const childrenRender = React.Children.map(children, child => {
            return React.cloneElement(child, { ...children.props, ...others, ...this._eventFunction });
        });
        return childrenRender;
    };
}
```
Cách dùng
```jsx
<ValidateWrapper actionId={EActionPoint.Delete} onClick={this.delete}>
    <button color="red">Delete</button>
</ValidateWrapper>
// Những action không cần validate thì ko cần truyề vào ValidateWrapper
<ValidateWrapper actionId={EActionPoint.ViewDetail} onClick={this.viewDetail}>
    <div onHover={this._onHover}>Xem chi tiết</div>
</ValidateWrapper>
```

<a name="basebutton"/>

##### 2.2 BaseButton
Một hệ thống luôn có một loạt các button dùng để thực hiện hành động gì đó, các button này thường chỉ khác nhau ở title và hành động, ngoài ra mỗi button đều có thể có shortcut cho hành động của nó.
VD: Xem trang http://link2.i3solution.net.au/
    - username: admin01
    - password: 123123Aa
    - vào tabs RESULTS, check chọn 1 vài item sẽ thấy một danh sách button hiện lên (nếu thấy ít button thì hãy select thêm các item khác để hiện thêm nút)
    
`BaseButton` sẽ là bộ khung cho các button đó, và tất nhiên, `BaseButton` cũng sẽ có luôn `ValidateWrapper` trong đó.
```jsx
class BaseButton extends BaseConsumer {
    constructor(props){
        super(props);
        // shortcut update sau
    }
    getActionId(){
        throw 'not implemented';
    }
    buttonTitle(){
        return null;
    }
    consumerContent(){
        <ValidateWrapper
            actionId={this.getActionId()}
            onClick={this.onClick.bind(this)}
        >
            <EHealthButton>
                {this.buttonTitle()}
            </EHealthButton>
        </ValidateWrapper>
    }
}
```
Implement một số button
```jsx
class DeleteButton extends BaseButton {
    getActionId(){
        return EActionPoint.Delele
    }
    buttonTitle() {
        return "Delete";
    }
    onClick() {
        // delete
    }
}
```
Sử dụng
```jsx
<DeleteButton {...parameters....}/>
```

<a name="field-validation"/>

##### 2.3 Field Validation
Ở Link2.0 đã có sẵn các class để thực hiện chuyện này, tuy nhiên khi áp dụng sang Ehealth chưa chắc đã đảm bảo đầy đủ chức năng, vì vậy trong quá trình sử dụng gặp vấn đề gì thì chúng ta sẽ bàn thêm để mở rộng, hiện những class này đã ở Link2.0 nhưng sẽ public lên Nuget sớm
Danh sách các Validator:
- `StringValidator`: Validate string
- `IntValidator`: Validate số Int
- `NIntValidator`: Validate số Int?
- `RangeValidator<T>`: Validate T có nằm trong các giá trị valid (select)
- `ObjectValidator<T>`: Validate một object T, class này sẽ có hàm `AddValidator`

các validator trên đều implement interface `IValidatable` đều có 2 phương thức
```csharp
public interface IValidatable<T>
{
    // nhận vào ack và "nhét" error message vào ack luôn
    bool IsValid(T value, Acknowledgement acknowledgement);
    
    // out ra message
    bool IsValid(T value, out string message);
}
```

Ví dụ có một class là `Patient` và cần validate trước khi thêm/xóa/sửa vào database và được validate dựa vào các tiêu chí như sau:
Parameter | Type | Min length | Max length | Min value | Max value | Valid values | Required
:--- | :--- | :--- | :--- | :---  | :--- | :--- | :---
Age | Number | N/A | N/A | 1 | 90 | N/A | No
Name | Text | 2 | 30 | N/A | N/A | N/A | Yes
Type | Range | N/A | N/A | N/A | N/A | 1, 2, 3 | Yes
```csharp
public class Patient
{
    public int Age { get; set; }
    public string Name { get; set; }
    public int Type { get; set; }
}
```
- Cách 1: Validate cả object, ta sẽ tạo `PatientValidator` như sau:
```csharp
public class PatientValidator : ObjectValidator<Patient>
{
    public PatientValidator() : base()
    {
        AddValidator(i => i.Age, new IntValidator()
        {
            IsRequired = false,
            MaxValue = 90,
            MinValue = 1,
            PropertyName = "Patient age",
        });
        AddValidator(i => i.Name, new StringValidator()
        {
            IsRequired = true,
            MaxLength = 30,
            MinLength = 2,
            PropertyName = "Patient name",
        });
        AddValidator(i => i.Type, new RangeValidator<int>(1, 2, 3)
        {
            IsRequired = true,
            PropertyName = "Patient age",
        });
    }
}
```
Cách dùng
```csharp
void validatePatient(Patient model)
{
    string message;
    var validator = new PatientValidator();
    bool check = validator.IsValid(model, out message);
    // check trả về false nếu model is invalid kèm theo message, nếu true thì message null
}
```


- Cách 2: Validate từng trường
```csharp
 public class PatientAgeValidator : IntValidator
{
    public PatientAgeValidator() : base()
    {
        IsRequired = false;
        MaxValue = 90;
        MinValue = 1;
        PropertyName = "Patient age";
    }
}
public class PatientNameValidator : StringValidator
{
    public PatientNameValidator() : base()
    {
        IsRequired = true;
        MaxLength = 30;
        MinLength = 2;
        PropertyName = "Patient name";
    }
}
public class PatientTypeValidator : RangeValidator<int>
{
    public PatientTypeValidator() : base(1, 2, 3)
    {
        IsRequired = true;
        PropertyName = "Patient age";
    }
}
```
Cách dùng
```csharp
void validatePatient(Patient model)
{
    string message;
    var ageValidator = new PatientAgeValidator();
    bool ageValid = ageValidator.IsValid(model.Age, out message);
    
    var nameValidator = new PatientNameValidator();
    bool nameValid = nameValidator.IsValid(model.Name, out message);
    
    var typeValidator = new PatientTypeValidator();
    bool typeValid = typeValidator.IsValid(model.Type, out message);
}
```


