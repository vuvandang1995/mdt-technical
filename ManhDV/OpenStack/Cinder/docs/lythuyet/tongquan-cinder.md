# Tổng quan về project Cinder

# 1. Giới thiệu


Cinder là dịch vụ Block Storage trong Openstack. Nó được thiết kế để người dùng cuối có thể thực hiện việc lưu trữ bởi Nova, việc này được thực hiện bởi LVM hoặc các plugin driver cho các nền tảng lưu trữ khác. Cinder ảo hóa việc quản lý các thiết bị block storage và cung cấp cho người dùng cuối một API đáp ứng được nhu cầu tự phục vụ cũng như tiêu thụ các tài nguyên đó mà không cần biết quá nhiều kiến thức chuyên sâu.

# 2. Kiến trúc và cơ chế hoạt động của Cinder

## 2.1. Kiến trúc của Cinder

![cinder](/ManhDV/Openstack/Cinder/images/cinder-achitecture.png)


 - cinder-api : Xác thực và định tuyến các yêu cầu xuyên suốt dịch vụ Cinder. 
 
 - cinder-scheduler : Lên lịch và định tuyến các yêu cầu tới dịch vụ volume thích hợp. Tùy thuộc vào cách cấu hình, có thể chỉ là dùng round-robin để định ra việc sẽ dùng volume service nào, hoặc có thể phức tạp hơn bằng cách dùng Filter Scheduler. Filter Scheduler là mặc định và bật các bộ lọc như Capacity, Avaibility Zone, Volume Type, và Capability.

 - cinder-volume : Quản lý thiết bị block storage, đặc biệt là các thiết bị back-end
 
 - cinder-backup : Cung cấp phương thức để backup một Block Storage volume tới Openstack Object Storage (Swift)
 
 - Driver : Chứa các mã back-end cụ thể để có thể liên lạc với các loại lưu trữ khác nhau.
 
 - Storage : Các thiết bị lưu trữ từ các nhà cung cấp khác nhau.

 - SQL DB : Cung cấp một phương tiện dùng để back up dữ liệu từ Swift/Celp, etc,....

 
## 2.2. Các thành phần có trong Cinder

 - Back-end Storage Device : Dịch vụ Block Storage yêu cầu một vài kiểu của back-end storage mà dịch vụ có thể chạy trên đó. Mặc định là sử dụng LVM trên một local volume group tên là "cinder-volumes"
 
 - User và Project : Cinder được dùng bởi các người dùng hoặc khách hàng khác nhau (project trong một shared system), sử dụng chỉ định truy cập dưa vào role (role-based access). Các role kiểm soát các hành động mà người dùng được phép thực hiện. Trong cấu hình mặc định, phần lớn các hành động không yêu cầu một role cụ thể, nhưng sysad có teher cấu hình trong file policy.json để quản lý các rule. Một truy cập của người dùng có thể bị hạn chế bởi project, nhưng username và pass được gán chỉ định cho mỗi user. Key pairs cho phép truy cập tới một volume được mở cho mỗi user, nhưng hạn ngạch để kiểm soát sự tiêu thu tài nguyên trên các tài nguyên phần cứng có sẵn là cho mỗi project.
 
 - Volume, Snapshot và Backup
	- Volume : Các tài nguyên block storage được phân phối có thể gán vào máy ảo như một ổ lưu trữ thứ 2 hoặc có thể dùng như là vùng lưu trữ cho root để boot máy ảo. Volume là các thiết bị block storage R/W bền vững thường được dùng để gán vào compute node thông qua iSCSI.
	
	- Snapshot : Một bản copy trong một thời điểm nhất định của một volume. Snapshot có thể được tạo từ một volume mà mới được dùng gaafnd ây trong trạng thái sẵn sàng. Snapshot có thể được dùng để tạo một volume mới thông qua việc tạo từ snapshot.
	
	- Backup : Một bản copy lưu trữ của một volume thông thường được lưu ở Swift.

## 2.3 Luồng làm việc của Cinder khi Attach Volume

	 - 1. Cinder gọi Cinder qua APi của cinder, truyền thông tin kết nối. Ví vụ : Host name, iSCSI initiator name, FC WWPNs
	 - 2. Cinder-API chuyển thông điệp đến Cinder-volume. Sau đó trình kiểm tra lỗi đầu vào sẽ làm việc và gọi đến volume driver.
	 - 3. Volume Driver sẽ chuẩn bị các yếu tố cần thiết để cho phép kết nối. Ví dụ : Cho phép máy chủ NOVA có thể truy cập vào Volume.
	 - 4. Volume driver trả về thông tin kết nối, được truyền cho NOVA. Ví dụ : iSCSI iqn and portal, FC WWPN.
	 - 5. NOVA tạo kết nối đến storage sử dụng thông tin được trả về.
	 - 6. NOVA chuyển volume device/file tới hypervisor.
	 
## 2.4 Các phương thức boot máy ảo (từ góc nhìn đối với Cinder)

Trong Openstack, có các cách khác nhau để tạo máy ảo là : 
	- Image : Tạo một ephameral disk từ image đã chọn
	- Volume : Boot máy ảo từ một bootable volume đã có sẵn
	- Image (tạo một volume mới) : Tạo một bootable volume mới từ image đã chọn và boot mấy ảo từ đó.
	- Volume snapshot (tạo một volume mới) : Tạo một volume từ volume snapshot đã chọn và boot máy ảo từ đó/
	
## 2.5 Điểm khác nhau giữa Ephemeral và Volume boot disk

### 2.5.1. Ephemeral Boot Disk

Ephemeral disk là disk ảo mà được tạo cho mục đích boot một máy ảo và nên được coi là nhất thời.

Ephemeral disk hữu dụng trong trường hợp bạn không lo lắng về nhu cầu nhân đôi một máy ảo hoặc hủy một máy ảo và dữ liệu trong đó sẽ mất hết. Bạn vẫn có thể mount một volume trên một máy ảo được boot từ một ephemeral disk và đẩy bất kỳ data nào cần thiết để lưu lại trong volume.