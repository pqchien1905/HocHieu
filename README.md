# Hướng Dẫn Hiểu Code Admin Controllers - CloudBox Laravel

## 📚 Mục Lục
1. [Giới Thiệu Cơ Bản](#giới-thiệu-cơ-bản)
2. [AdminFilesController - Quản Lý Files](#adminfilescontroller)
3. [AdminFoldersController - Quản Lý Folders](#adminfolderscontroller)
4. [AdminFavoritesController - Quản Lý Favorites](#adminfavoritescontroller)
5. [Các Khái Niệm Quan Trọng](#các-khái-niệm-quan-trọng)

---

## Giới Thiệu Cơ Bản

### Controller Là Gì?

**Controller** giống như một "người điều phối" trong ứng dụng web:
- Nhận yêu cầu từ người dùng (qua trình duyệt)
- Xử lý logic nghiệp vụ
- Lấy dữ liệu từ database
- Trả về kết quả (hiển thị view hoặc chuyển trang)

### Cấu Trúc File Controller

```php
<?php

namespace App\Http\Controllers\Admin;  // Khai báo namespace (địa chỉ file)

use App\Http\Controllers\Controller;   // Sử dụng class Controller gốc
use App\Models\File;                   // Sử dụng Model File

class AdminFilesController extends Controller  // Kế thừa từ Controller
{
    public function index() { }  // Method xử lý danh sách
    public function show() { }   // Method xử lý chi tiết
    // ...
}
```

---

## AdminFilesController

### 📝 Mục Đích
Controller này quản lý **tất cả các file** trong hệ thống từ phía Admin.

### 🔍 Method `index()` - Hiển Thị Danh Sách Files

```php
public function index(Request $request)
```

**Giải thích:**
- `Request $request`: Nhận tất cả dữ liệu từ URL (tìm kiếm, lọc, phân trang...)

#### 1. Khởi Tạo Query

```php
$query = File::with(['user', 'folder']);
```

**Giải thích:**
- `File::`: Gọi Model File (đại diện cho bảng `files` trong database)
- `with(['user', 'folder'])`: Eager Loading - Lấy luôn thông tin user và folder liên quan (tối ưu hiệu suất)

#### 2. Tìm Kiếm

```php
if ($request->filled('search')) {
    $search = $request->search;
    $query->where(function($q) use ($search) {
        $q->where('name', 'like', "%{$search}%")
          ->orWhere('original_name', 'like', "%{$search}%")
          ->orWhere('extension', 'like', "%{$search}%");
    });
}
```

**Giải thích:**
- `$request->filled('search')`: Kiểm tra xem có nhập từ khóa tìm kiếm không
- `where('name', 'like', "%{$search}%")`: Tìm kiếm gần đúng trong tên file
  - `like`: Toán tử so sánh tương đối
  - `%{$search}%`: % là ký tự đại diện (wildcard) - tìm chuỗi chứa từ khóa
- `orWhere()`: HOẶC - tìm trong nhiều cột

**Ví dụ:**
- Tìm "test" → Tìm file có tên: "test.pdf", "mytest.jpg", "testing.doc"

#### 3. Lọc Theo Người Dùng

```php
if ($request->filled('user_id')) {
    $query->where('user_id', $request->user_id);
}
```

**Giải thích:**
- Chỉ hiển thị file của user có ID cụ thể

#### 4. Lọc Theo Loại File

```php
if ($request->filled('type')) {
    $type = $request->type;
    switch ($type) {
        case 'image':
            $query->where('mime_type', 'like', 'image%');
            break;
        case 'video':
            $query->where('mime_type', 'like', 'video%');
            break;
        // ...
    }
}
```

**Giải thích:**
- `switch`: Câu lệnh rẽ nhánh nhiều điều kiện
- `mime_type`: Định dạng file (image/jpeg, video/mp4, application/pdf...)
- `image%`: Tìm tất cả file có mime_type bắt đầu bằng "image"

**MIME Type là gì?**
- `image/jpeg`: File ảnh JPEG
- `video/mp4`: File video MP4
- `application/pdf`: File PDF

#### 5. Lọc Theo Category

```php
if ($request->filled('category')) {
    $category = Category::where('slug', $request->category)
                        ->where('is_active', true)
                        ->first();
    if ($category && !empty($category->extensions)) {
        $query->whereIn('extension', $category->extensions);
    }
}
```

**Giải thích:**
- Tìm category theo slug (URL thân thiện)
- `first()`: Lấy 1 bản ghi đầu tiên
- `whereIn()`: Tìm file có extension nằm trong danh sách extensions của category

#### 6. Lọc Theo Trạng Thái

```php
if ($request->filled('status')) {
    if ($request->status === 'trash') {
        $query->where('is_trash', true);
    } elseif ($request->status === 'active') {
        $query->where('is_trash', false);
    }
}
```

**Giải thích:**
- `is_trash`: Cột boolean (true/false) đánh dấu file đã xóa
- `===`: So sánh chính xác giá trị và kiểu dữ liệu

#### 7. Lọc Theo Kích Thước

```php
if ($request->filled('size_min')) {
    $query->where('size', '>=', $request->size_min * 1024 * 1024);
}
```

**Giải thích:**
- `size_min * 1024 * 1024`: Chuyển đổi MB sang Bytes
  - 1 MB = 1024 KB
  - 1 KB = 1024 Bytes
  - 1 MB = 1024 × 1024 = 1,048,576 Bytes

#### 8. Sắp Xếp

```php
$sortBy = $request->get('sort', 'created_at');
$sortOrder = $request->get('order', 'desc');

$allowedSorts = ['name', 'size', 'created_at', 'updated_at'];
if (in_array($sortBy, $allowedSorts)) {
    $query->orderBy($sortBy, $sortOrder);
}
```

**Giải thích:**
- `get('sort', 'created_at')`: Lấy giá trị sort, nếu không có thì mặc định 'created_at'
- `in_array()`: Kiểm tra giá trị có trong mảng cho phép không (bảo mật)
- `orderBy()`: Sắp xếp kết quả
  - `desc`: Giảm dần (mới nhất trước)
  - `asc`: Tăng dần (cũ nhất trước)

#### 9. Phân Trang

```php
$files = $query->paginate(20)->withQueryString();
```

**Giải thích:**
- `paginate(20)`: Chia kết quả thành nhiều trang, mỗi trang 20 item
- `withQueryString()`: Giữ lại các tham số tìm kiếm/lọc khi chuyển trang

#### 10. Thống Kê

```php
$stats = [
    'total' => File::count(),
    'active' => File::where('is_trash', false)->count(),
    'trash' => File::where('is_trash', true)->count(),
    // ...
];
```

**Giải thích:**
- `count()`: Đếm số lượng bản ghi
- Tạo mảng thống kê để hiển thị tổng quan

#### 11. Trả Về View

```php
return view('pages.admin.files.index', compact('files', 'stats', 'users', 'categories'));
```

**Giải thích:**
- `view()`: Hiển thị file Blade template
- `compact()`: Truyền biến vào view
- Tương đương: `['files' => $files, 'stats' => $stats, ...]`

---

### 🔍 Method `show()` - Hiển Thị Chi Tiết File

```php
public function show(File $file)
{
    $file->load(['user', 'folder', 'shares.sharedBy', 'shares.sharedWith']);
    return view('pages.admin.files.show', compact('file'));
}
```

**Giải thích:**
- `File $file`: Route Model Binding - Laravel tự động tìm file theo ID từ URL
- `load()`: Lazy Loading - Load thêm quan hệ sau khi đã có object
- `shares.sharedBy`: Nested relationship - Lấy thông tin chia sẻ và người chia sẻ

**Route Model Binding là gì?**
```php
// Route: /admin/files/123
// Laravel tự động: File::findOrFail(123)
```

---

### 📥 Method `serve()` - Phục Vụ File

```php
public function serve(File $file)
{
    $filePath = storage_path('app/public/' . $file->path);
    
    if (!file_exists($filePath)) {
        abort(404, __('common.file_not_found'));
    }
    
    return response()->file($filePath, [
        'Content-Type' => $file->mime_type,
        'Content-Disposition' => 'inline; filename="' . $file->original_name . '"',
    ]);
}
```

**Giải thích:**
- `storage_path()`: Lấy đường dẫn đầy đủ đến thư mục storage
- `file_exists()`: Kiểm tra file có tồn tại không
- `abort(404)`: Trả về lỗi 404 Not Found
- `response()->file()`: Trả về nội dung file
  - `Content-Type`: Loại file (để trình duyệt biết cách hiển thị)
  - `Content-Disposition: inline`: Hiển thị trực tiếp trên trình duyệt (không tải xuống)

---

### 👁️ Method `view()` - Xem File Inline

```php
public function view(File $file)
{
    $filePath = storage_path('app/public/' . $file->path);
    
    if (!file_exists($filePath) && !Storage::disk('public')->exists($file->path)) {
        return redirect()->back()->with('error', __('common.file_not_found'));
    }
    
    $fileUrl = route('admin.files.serve', $file->id);
    
    return view('pages.admin.files.view', [
        'file' => $file,
        'fileUrl' => $fileUrl,
    ]);
}
```

**Giải thích:**
- `Storage::disk('public')`: Sử dụng Laravel Storage facade
- `redirect()->back()`: Quay lại trang trước
- `with('error', 'message')`: Flash message - Thông báo tạm thời
- `route()`: Tạo URL từ tên route

---

### ⬇️ Method `download()` - Tải File

```php
public function download(File $file)
{
    $filePath = storage_path('app/public/' . $file->path);
    
    if (!file_exists($filePath)) {
        return redirect()->back()->with('error', __('common.file_not_found'));
    }
    
    return response()->download($filePath, $file->original_name);
}
```

**Giải thích:**
- `response()->download()`: Tải file xuống máy
- Tham số thứ 2: Tên file khi tải (người dùng thấy)

---

### 🗑️ Method `destroy()` - Xóa File

```php
public function destroy(File $file)
{
    try {
        // Xóa file vật lý
        if ($file->path && Storage::disk('public')->exists($file->path)) {
            Storage::disk('public')->delete($file->path);
        }
        
        // Xóa record trong database
        $file->delete();
        
        return redirect()->route('admin.files.index')
            ->with('status', __('common.file_deleted_successfully'));
    } catch (\Exception $e) {
        return redirect()->back()
            ->with('error', __('common.error_deleting_file') . ': ' . $e->getMessage());
    }
}
```

**Giải thích:**
- `try...catch`: Xử lý lỗi
  - `try`: Thực thi code
  - `catch`: Bắt lỗi nếu có
- `Storage::delete()`: Xóa file vật lý khỏi ổ cứng
- `$file->delete()`: Xóa record khỏi database
- `\Exception $e`: Biến chứa thông tin lỗi
- `$e->getMessage()`: Lấy thông báo lỗi

**Tại sao phải xóa 2 lần?**
- File vật lý: File thực tế trên ổ cứng
- Database record: Thông tin về file trong database
- Cần xóa cả 2 để hoàn toàn xóa file

---

## AdminFoldersController

### 📝 Mục Đích
Controller này quản lý **tất cả các folder** trong hệ thống từ phía Admin.

### 🔍 Method `index()` - Hiển Thị Danh Sách Folders

```php
$query = Folder::with(['user', 'parent'])
    ->withCount(['files' => function ($q) {
        $q->where('is_trash', false);
    }])
    ->withCount('children');
```

**Giải thích:**
- `withCount()`: Đếm số lượng quan hệ
  - `withCount('files')`: Đếm số file trong folder → tạo cột `files_count`
  - `withCount('children')`: Đếm số folder con → tạo cột `children_count`
- `function ($q)`: Closure - Hàm ẩn danh để thêm điều kiện
  - Chỉ đếm file chưa bị xóa

**Ví dụ:**
```php
$folder->files_count;  // 15 files
$folder->children_count;  // 3 subfolders
```

### Lọc Theo Level (Cấp Độ)

```php
if ($request->filled('level')) {
    if ($request->level === 'root') {
        $query->whereNull('parent_id');  // Folder gốc
    } elseif ($request->level === 'subfolder') {
        $query->whereNotNull('parent_id');  // Folder con
    }
}
```

**Giải thích:**
- `whereNull('parent_id')`: Folder không có cha → Folder gốc
- `whereNotNull('parent_id')`: Folder có cha → Folder con

**Cấu trúc Folder:**
```
📁 Documents (parent_id = null) ← ROOT
  📁 Work (parent_id = Documents.id) ← SUBFOLDER
    📁 Projects (parent_id = Work.id) ← SUBFOLDER
  📁 Personal (parent_id = Documents.id) ← SUBFOLDER
```

---

### 🔍 Method `show()` - Hiển Thị Chi Tiết Folder

```php
public function show(Folder $folder)
{
    $folder->load(['user', 'parent', 'children' => function($query) {
        $query->where('is_trash', false)->withCount(['files' => function($q) {
            $q->where('is_trash', false);
        }]);
    }, 'files' => function($query) {
        $query->where('is_trash', false);
    }]);
    
    $totalSize = $folder->files()->where('is_trash', false)->sum('size');
    
    return view('pages.admin.folders.show', compact('folder', 'totalSize'));
}
```

**Giải thích:**
- Load nhiều quan hệ với điều kiện phức tạp
- `sum('size')`: Tính tổng kích thước tất cả file

**Ví dụ:**
```php
// Folder có 3 files: 100KB, 200KB, 300KB
$totalSize = 600KB
```

---

### ⬇️ Method `download()` - Tải Folder Dưới Dạng ZIP

```php
public function download(Folder $folder)
{
    try {
        $zipFileName = Str::slug($folder->name) . '_' . time() . '.zip';
        $zipFilePath = storage_path('app/temp/' . $zipFileName);
        
        if (!file_exists(storage_path('app/temp'))) {
            mkdir(storage_path('app/temp'), 0755, true);
        }
        
        $zip = new \ZipArchive();
        if ($zip->open($zipFilePath, \ZipArchive::CREATE | \ZipArchive::OVERWRITE) !== true) {
            return redirect()->back()->with('error', __('common.error_creating_zip'));
        }
        
        $this->addFolderToZip($zip, $folder, $folder->name);
        
        $zip->close();
        
        return response()->download($zipFilePath, $zipFileName)->deleteFileAfterSend(true);
    } catch (\Exception $e) {
        return redirect()->back()->with('error', __('common.error_downloading_folder') . ': ' . $e->getMessage());
    }
}
```

**Giải thích:**

#### 1. Tạo Tên File ZIP
```php
$zipFileName = Str::slug($folder->name) . '_' . time() . '.zip';
```
- `Str::slug()`: Chuyển tên thành URL-friendly
  - "My Documents" → "my-documents"
- `time()`: Thêm timestamp để tránh trùng tên

#### 2. Tạo Thư Mục Temp
```php
if (!file_exists(storage_path('app/temp'))) {
    mkdir(storage_path('app/temp'), 0755, true);
}
```
- `mkdir()`: Tạo thư mục
- `0755`: Quyền truy cập (owner: read/write/execute, others: read/execute)
- `true`: Tạo cả thư mục cha nếu chưa tồn tại

#### 3. Tạo File ZIP
```php
$zip = new \ZipArchive();
$zip->open($zipFilePath, \ZipArchive::CREATE | \ZipArchive::OVERWRITE);
```
- `\ZipArchive`: Class PHP để làm việc với ZIP
- `CREATE`: Tạo file mới
- `OVERWRITE`: Ghi đè nếu đã tồn tại

#### 4. Thêm File Vào ZIP
```php
$this->addFolderToZip($zip, $folder, $folder->name);
```
- Gọi method đệ quy để thêm tất cả file và folder con

#### 5. Trả Về File ZIP
```php
return response()->download($zipFilePath, $zipFileName)->deleteFileAfterSend(true);
```
- `deleteFileAfterSend(true)`: Tự động xóa file tạm sau khi tải xong

---

### 🔄 Method `addFolderToZip()` - Thêm Folder Vào ZIP (Đệ Quy)

```php
private function addFolderToZip(\ZipArchive $zip, Folder $folder, string $basePath)
{
    // Thêm tất cả file trong folder
    foreach ($folder->files()->where('is_trash', false)->get() as $file) {
        $filePath = storage_path('app/public/' . $file->path);
        if (file_exists($filePath)) {
            $zip->addFile($filePath, $basePath . '/' . $file->original_name);
        }
    }
    
    // Thêm tất cả folder con (đệ quy)
    foreach ($folder->children()->where('is_trash', false)->get() as $childFolder) {
        $this->addFolderToZip($zip, $childFolder, $basePath . '/' . $childFolder->name);
    }
}
```

**Giải thích:**

#### Đệ Quy Là Gì?
- Method tự gọi chính nó
- Dùng để xử lý cấu trúc cây (folder có folder con có folder con...)

**Ví dụ Cấu Trúc:**
```
Documents/
  ├── file1.pdf
  ├── Work/
  │   ├── file2.docx
  │   └── Projects/
  │       └── file3.xlsx
  └── Personal/
      └── file4.jpg
```

**Cách Hoạt Động:**
1. addFolderToZip(Documents, "Documents")
   - Thêm file1.pdf → "Documents/file1.pdf"
   - Gọi addFolderToZip(Work, "Documents/Work")
     - Thêm file2.docx → "Documents/Work/file2.docx"
     - Gọi addFolderToZip(Projects, "Documents/Work/Projects")
       - Thêm file3.xlsx → "Documents/Work/Projects/file3.xlsx"
   - Gọi addFolderToZip(Personal, "Documents/Personal")
     - Thêm file4.jpg → "Documents/Personal/file4.jpg"

---

### 🗑️ Method `destroy()` - Xóa Folder

```php
public function destroy(Folder $folder)
{
    try {
        // Xóa tất cả file trong folder
        foreach ($folder->files as $file) {
            if ($file->path && Storage::disk('public')->exists($file->path)) {
                Storage::disk('public')->delete($file->path);
            }
            $file->delete();
        }
        
        // Xóa tất cả folder con (recursive)
        $this->deleteFolderRecursive($folder);
        
        // Xóa record
        $folder->delete();
        
        return redirect()->route('admin.folders.index')
            ->with('status', __('common.folder_deleted_successfully'));
    } catch (\Exception $e) {
        return redirect()->back()
            ->with('error', __('common.error_deleting_folder') . ': ' . $e->getMessage());
    }
}
```

**Giải thích:**
- Xóa theo thứ tự:
  1. File trong folder hiện tại
  2. Tất cả folder con (đệ quy)
  3. Folder hiện tại

### 🔄 Method `deleteFolderRecursive()` - Xóa Đệ Quy

```php
private function deleteFolderRecursive(Folder $folder)
{
    foreach ($folder->children as $child) {
        // Xóa file trong folder con
        foreach ($child->files as $file) {
            if ($file->path && Storage::disk('public')->exists($file->path)) {
                Storage::disk('public')->delete($file->path);
            }
            $file->delete();
        }
        
        // Đệ quy xóa folder con
        $this->deleteFolderRecursive($child);
        
        // Xóa folder con
        $child->delete();
    }
}
```

**Giải thích:**
- Duyệt qua từng folder con
- Xóa file trong folder con
- Gọi lại chính nó để xóa folder con của folder con
- Cuối cùng xóa folder con

**Tại sao cần đệ quy?**
- Folder có thể có nhiều cấp con
- Phải xóa từ trong ra ngoài

---

## AdminFavoritesController

### 📝 Mục Đích
Controller này quản lý **tất cả các item yêu thích** (cả files và folders) trong hệ thống.

### 🔍 Method `index()` - Hiển Thị Danh Sách Favorites

```php
public function index(Request $request)
{
    // Query favorite files
    $filesQuery = File::with(['user', 'folder'])
        ->where('is_favorite', true);
    
    // Query favorite folders
    $foldersQuery = Folder::with(['user', 'parent'])
        ->where('is_favorite', true);
    
    // Filter by user
    if ($request->filled('user_id')) {
        $filesQuery->where('user_id', $request->user_id);
        $foldersQuery->where('user_id', $request->user_id);
    }
    
    // Filter by type
    $type = $request->get('type', 'all');
    
    $files = collect();
    $folders = collect();
    
    if ($type === 'all' || $type === 'files') {
        $files = $filesQuery->get();
    }
    
    if ($type === 'all' || $type === 'folders') {
        $folders = $foldersQuery->get();
    }
    
    // ...
}
```

**Giải thích:**
- Query riêng files và folders
- `collect()`: Tạo Collection rỗng
- Lấy dữ liệu dựa trên type filter

#### Collection Là Gì?
- Đối tượng để làm việc với mảng dữ liệu
- Có nhiều method tiện ích: `map()`, `filter()`, `sortBy()`, `sum()`...

---

### 🔀 Gộp Files và Folders

```php
$items = collect();

foreach ($folders as $folder) {
    $items->push([
        'type' => 'folder',
        'id' => $folder->id,
        'name' => $folder->name,
        'user' => $folder->user,
        'parent' => $folder->parent,
        'size' => null,
        'created_at' => $folder->created_at,
        'model' => $folder,
    ]);
}

foreach ($files as $file) {
    $items->push([
        'type' => 'file',
        'id' => $file->id,
        'name' => $file->name,
        'original_name' => $file->original_name,
        'user' => $file->user,
        'folder' => $file->folder,
        'size' => $file->size,
        'formatted_size' => $file->formatted_size,
        'mime_type' => $file->mime_type,
        'extension' => $file->extension,
        'created_at' => $file->created_at,
        'model' => $file,
    ]);
}
```

**Giải thích:**
- Tạo mảng thống nhất cho cả files và folders
- Thêm trường `type` để phân biệt
- `push()`: Thêm item vào Collection

**Tại sao cần gộp?**
- Để hiển thị cả files và folders trong 1 danh sách
- Để sort chung theo created_at, name...

---

### 📄 Phân Trang Thủ Công

```php
// Sort combined items
if ($sortOrder === 'desc') {
    $items = $items->sortByDesc($sortBy);
} else {
    $items = $items->sortBy($sortBy);
}

// Paginate manually
$perPage = 20;
$currentPage = $request->get('page', 1);
$total = $items->count();
$items = $items->forPage($currentPage, $perPage);

$paginator = new \Illuminate\Pagination\LengthAwarePaginator(
    $items,
    $total,
    $perPage,
    $currentPage,
    ['path' => $request->url(), 'query' => $request->query()]
);
```

**Giải thích:**

#### 1. Sort Collection
```php
$items = $items->sortByDesc($sortBy);
```
- `sortBy()`: Sắp xếp tăng dần
- `sortByDesc()`: Sắp xếp giảm dần

#### 2. Phân Trang
```php
$items = $items->forPage($currentPage, $perPage);
```
- `forPage(1, 20)`: Lấy 20 item đầu tiên (trang 1)
- `forPage(2, 20)`: Lấy 20 item tiếp theo (trang 2)

#### 3. Tạo Paginator
```php
new \Illuminate\Pagination\LengthAwarePaginator(...)
```
- Tạo object phân trang giống như `paginate()`
- `LengthAwarePaginator`: Biết tổng số item

**Tại sao phân trang thủ công?**
- Query từ 2 bảng khác nhau (files, folders)
- Không thể dùng `paginate()` trực tiếp

---

### ❤️ Method `unfavoriteFile()` - Bỏ Yêu Thích File

```php
public function unfavoriteFile(Request $request, File $file)
{
    try {
        $file->update(['is_favorite' => false]);
        
        return redirect()->route('admin.favorites.index', $request->query())
            ->with('status', __('common.file_removed_from_favorites'));
    } catch (\Exception $e) {
        return redirect()->route('admin.favorites.index', $request->query())
            ->with('error', __('common.error_removing_favorite') . ': ' . $e->getMessage());
    }
}
```

**Giải thích:**
- `update()`: Cập nhật record
- `$request->query()`: Lấy tất cả query parameters
  - Giữ lại filter, search, page khi redirect

**Ví dụ:**
```
URL trước: /admin/favorites?page=2&search=test
Sau unfavorite: vẫn về /admin/favorites?page=2&search=test
```

---

### 🗑️ Method `bulkUnfavorite()` - Bỏ Yêu Thích Hàng Loạt

```php
public function bulkUnfavorite(Request $request)
{
    $request->validate([
        'file_ids' => 'array',
        'file_ids.*' => 'exists:files,id',
        'folder_ids' => 'array',
        'folder_ids.*' => 'exists:folders,id',
    ]);
    
    $count = 0;
    
    if ($request->filled('file_ids')) {
        $count += File::whereIn('id', $request->file_ids)->update(['is_favorite' => false]);
    }
    
    if ($request->filled('folder_ids')) {
        $count += Folder::whereIn('id', $request->folder_ids)->update(['is_favorite' => false]);
    }
    
    return redirect()->route('admin.favorites.index')
        ->with('status', __('common.items_removed_from_favorites', ['count' => $count]));
}
```

**Giải thích:**

#### 1. Validation
```php
$request->validate([
    'file_ids' => 'array',
    'file_ids.*' => 'exists:files,id',
]);
```
- `'array'`: file_ids phải là mảng
- `'file_ids.*'`: Mỗi phần tử trong mảng
- `'exists:files,id'`: ID phải tồn tại trong bảng files

#### 2. Update Hàng Loạt
```php
File::whereIn('id', $request->file_ids)->update(['is_favorite' => false]);
```
- `whereIn()`: WHERE id IN (1, 2, 3, ...)
- `update()`: Cập nhật tất cả record khớp
- Trả về số record đã cập nhật

**Ví dụ:**
```php
// Request: file_ids = [1, 5, 10]
// SQL: UPDATE files SET is_favorite = 0 WHERE id IN (1, 5, 10)
// Trả về: 3 (số file đã cập nhật)
```

---

## Các Khái Niệm Quan Trọng

### 1. 🏗️ MVC Pattern (Model-View-Controller)

```
User Request → Route → Controller → Model → Database
                           ↓
                        View ← Response
```

**Vai trò:**
- **Model**: Làm việc với database
- **View**: Hiển thị giao diện
- **Controller**: Điều phối logic nghiệp vụ

---

### 2. 🔗 Eloquent Relationships

#### One-to-Many (Một-Nhiều)
```php
// User có nhiều File
class User extends Model {
    public function files() {
        return $this->hasMany(File::class);
    }
}

class File extends Model {
    public function user() {
        return $this->belongsTo(User::class);
    }
}
```

**Sử dụng:**
```php
$user->files;  // Lấy tất cả file của user
$file->user;   // Lấy user sở hữu file
```

#### Self-Referencing (Tự Tham Chiếu)
```php
class Folder extends Model {
    public function parent() {
        return $this->belongsTo(Folder::class, 'parent_id');
    }
    
    public function children() {
        return $this->hasMany(Folder::class, 'parent_id');
    }
}
```

**Sử dụng:**
```php
$folder->parent;    // Folder cha
$folder->children;  // Các folder con
```

---

### 3. 🚀 Eager Loading vs Lazy Loading

#### Lazy Loading (Tải Chậm)
```php
$files = File::all();
foreach ($files as $file) {
    echo $file->user->name;  // Query mỗi lần lặp
}
// 1 + N queries (N+1 problem)
```

#### Eager Loading (Tải Sớm)
```php
$files = File::with('user')->all();
foreach ($files as $file) {
    echo $file->user->name;  // Đã có sẵn
}
// Chỉ 2 queries
```

**Tại sao nên dùng Eager Loading?**
- Giảm số lượng query
- Tăng hiệu suất
- Tránh N+1 problem

---

### 4. 📝 Query Builder

#### Method Chaining
```php
File::where('user_id', 1)
    ->where('is_trash', false)
    ->orderBy('created_at', 'desc')
    ->limit(10)
    ->get();
```

**Giải thích:**
- Mỗi method trả về chính nó → có thể gọi tiếp method khác
- `get()`: Thực thi query và lấy kết quả

#### Các Method Hay Dùng

| Method | Mô Tả | Ví Dụ |
|--------|-------|-------|
| `where()` | Điều kiện WHERE | `where('size', '>', 1000)` |
| `orWhere()` | Điều kiện OR | `orWhere('type', 'image')` |
| `whereIn()` | WHERE IN | `whereIn('id', [1,2,3])` |
| `whereNull()` | WHERE IS NULL | `whereNull('deleted_at')` |
| `orderBy()` | Sắp xếp | `orderBy('name', 'asc')` |
| `limit()` | Giới hạn số lượng | `limit(10)` |
| `get()` | Lấy kết quả | `get()` |
| `first()` | Lấy 1 kết quả | `first()` |
| `count()` | Đếm | `count()` |
| `sum()` | Tổng | `sum('size')` |
| `paginate()` | Phân trang | `paginate(20)` |

---

### 5. 🛡️ Validation

```php
$request->validate([
    'name' => 'required|string|max:255',
    'email' => 'required|email|unique:users',
    'age' => 'nullable|integer|min:18',
    'file' => 'required|file|mimes:pdf,jpg|max:10240',
]);
```

**Các Rule Phổ Biến:**
- `required`: Bắt buộc
- `nullable`: Có thể null
- `string`: Kiểu chuỗi
- `integer`: Kiểu số nguyên
- `email`: Định dạng email
- `max:255`: Tối đa 255 ký tự
- `min:18`: Tối thiểu 18
- `unique:users`: Duy nhất trong bảng users
- `exists:users,id`: Tồn tại trong bảng users, cột id
- `mimes:pdf,jpg`: Loại file
- `file`: Phải là file

---

### 6. 🔐 Security Best Practices

#### 1. Mass Assignment Protection
```php
class File extends Model {
    protected $fillable = ['name', 'path', 'size'];  // Cho phép gán hàng loạt
    protected $guarded = ['id', 'user_id'];         // Bảo vệ không cho gán
}
```

#### 2. Input Validation
```php
// ✅ TỐT
$request->validate(['type' => 'required|in:image,video,pdf']);

// ❌ XẤU
$type = $request->type;  // Không validate → SQL injection risk
```

#### 3. SQL Injection Prevention
```php
// ✅ TỐT - Query Builder tự động escape
File::where('name', $request->search)->get();

// ❌ XẤU - Raw query không escape
DB::select("SELECT * FROM files WHERE name = '{$request->search}'");
```

#### 4. XSS Prevention
```blade
{{-- ✅ TỐT - Laravel tự động escape --}}
{{ $file->name }}

{{-- ❌ XẤU - Không escape --}}
{!! $file->name !!}
```

---

### 7. 📦 Storage và Filesystem

#### Storage Disks
```php
// Public disk: storage/app/public
Storage::disk('public')->put('file.pdf', $content);

// Local disk: storage/app
Storage::disk('local')->put('private.pdf', $content);
```

#### Các Method Hay Dùng
```php
Storage::exists('file.pdf');           // Kiểm tra tồn tại
Storage::get('file.pdf');              // Đọc nội dung
Storage::put('file.pdf', $content);    // Lưu file
Storage::delete('file.pdf');           // Xóa file
Storage::size('file.pdf');             // Kích thước
Storage::lastModified('file.pdf');     // Thời gian sửa cuối
```

---

### 8. 🌐 Response Types

```php
// Trả về view
return view('admin.files.index', compact('files'));

// Redirect
return redirect()->route('admin.files.index');
return redirect()->back();

// JSON response
return response()->json(['data' => $files]);

// File response
return response()->file($path);
return response()->download($path, 'filename.pdf');

// With flash message
return redirect()->back()->with('status', 'Success!');
```

---

### 9. 🎯 Route Model Binding

```php
// Route
Route::get('/files/{file}', [AdminFilesController::class, 'show']);

// Controller
public function show(File $file) {
    // Laravel tự động: File::findOrFail($id)
    // Nếu không tìm thấy → 404
}
```

**Lợi ích:**
- Code ngắn gọn hơn
- Tự động xử lý 404
- Type hinting (IDE autocomplete)

---

### 10. 📊 Collections

```php
$collection = collect([1, 2, 3, 4, 5]);

$collection->map(fn($item) => $item * 2);     // [2, 4, 6, 8, 10]
$collection->filter(fn($item) => $item > 3);  // [4, 5]
$collection->sum();                            // 15
$collection->count();                          // 5
$collection->first();                          // 1
$collection->last();                           // 5
$collection->push(6);                          // [1, 2, 3, 4, 5, 6]
$collection->pluck('name');                    // Lấy cột 'name'
$collection->sortBy('created_at');             // Sắp xếp
$collection->groupBy('type');                  // Nhóm theo type
```

---

## 🎓 Luồng Hoạt Động Tổng Thể

### Ví Dụ: User Xem Danh Sách Files

```
1. User truy cập: http://example.com/admin/files

2. Route (web.php):
   Route::get('/admin/files', [AdminFilesController::class, 'index']);

3. Controller (AdminFilesController):
   public function index(Request $request) {
       $files = File::with('user')->paginate(20);
       return view('pages.admin.files.index', compact('files'));
   }

4. Model (File):
   - Lấy dữ liệu từ bảng 'files'
   - Load quan hệ 'user'

5. View (index.blade.php):
   - Nhận biến $files
   - Hiển thị danh sách
   - Render HTML

6. Response:
   - HTML được trả về trình duyệt
   - User thấy danh sách files
```

---

## 💡 Tips Học Laravel

### 1. Đọc Code Từ Route
- Bắt đầu từ file `routes/web.php`
- Tìm route tương ứng
- Theo dõi vào Controller
- Xem View được render

### 2. Sử dụng Tinker
```bash
php artisan tinker

>>> $file = File::first();
>>> $file->user;
>>> $file->user->name;
```

### 3. Debug Queries
```php
DB::enableQueryLog();
File::where('user_id', 1)->get();
dd(DB::getQueryLog());
```

### 4. Đọc Documentation
- Laravel Docs: https://laravel.com/docs
- Tìm kiếm trên Google với từ khóa "Laravel [method name]"

---

## 📚 Thuật Ngữ Quan Trọng

| Thuật Ngữ | Tiếng Việt | Giải Thích |
|-----------|------------|------------|
| Controller | Bộ điều khiển | Xử lý logic nghiệp vụ |
| Model | Mô hình | Đại diện cho bảng database |
| View | Giao diện | File template hiển thị |
| Route | Định tuyến | Ánh xạ URL → Controller |
| Migration | Di chuyển | File tạo/sửa cấu trúc database |
| Eloquent | - | ORM của Laravel |
| Query Builder | Trình xây dựng truy vấn | Tạo SQL bằng PHP |
| Middleware | Lớp trung gian | Xử lý request trước khi vào Controller |
| Validation | Kiểm tra hợp lệ | Kiểm tra dữ liệu đầu vào |
| Eager Loading | Tải sớm | Load quan hệ cùng lúc |
| Lazy Loading | Tải chậm | Load quan hệ khi cần |
| Route Model Binding | - | Tự động load Model từ URL |
| Mass Assignment | Gán hàng loạt | Gán nhiều thuộc tính cùng lúc |
| Collection | Tập hợp | Đối tượng làm việc với mảng |
| Pagination | Phân trang | Chia kết quả thành nhiều trang |

---

## ✅ Checklist Kiến Thức

Sau khi đọc tài liệu này, bạn nên hiểu:

- [ ] Controller là gì và vai trò của nó
- [ ] Các method trong Controller (index, show, store, update, destroy)
- [ ] Query Builder và các method cơ bản
- [ ] Eloquent Relationships
- [ ] Eager Loading vs Lazy Loading
- [ ] Validation
- [ ] Storage và Filesystem
- [ ] Response types
- [ ] Route Model Binding
- [ ] Collections
- [ ] Try-catch và xử lý lỗi
- [ ] Đệ quy (Recursion)

---

## 🚀 Bước Tiếp Theo

1. **Thực hành**: Tạo một Controller đơn giản
2. **Đọc code**: Xem các Controller khác trong project
3. **Thử nghiệm**: Sửa đổi code và xem kết quả
4. **Debug**: Sử dụng `dd()`, `dump()` để debug
5. **Đọc docs**: Laravel documentation
6. **Xây dựng**: Tạo feature mới

---

## 📖 Tài Liệu Tham Khảo

- [Laravel Documentation](https://laravel.com/docs)
- [Eloquent ORM](https://laravel.com/docs/eloquent)
- [Query Builder](https://laravel.com/docs/queries)
- [Validation](https://laravel.com/docs/validation)
- [Collections](https://laravel.com/docs/collections)
- [File Storage](https://laravel.com/docs/filesystem)

---

---

## 🎨 Hiểu Về Blade Views

### View Là Gì?

**View** (hay Template) là file chứa mã HTML để hiển thị giao diện cho người dùng:
- Nhận dữ liệu từ Controller
- Hiển thị dữ liệu dưới dạng HTML
- Sử dụng Blade template engine của Laravel

### Blade Template Engine

**Blade** là công cụ template của Laravel giúp viết HTML dễ dàng hơn:
- Cú pháp đơn giản
- Kế thừa layout
- Hiển thị dữ liệu an toàn (tự động escape XSS)
- Các directive tiện ích (@if, @foreach, @auth...)

---

## 📄 AdminFilesController Views

### 1. `index.blade.php` - Danh Sách Files

#### Cấu Trúc Cơ Bản

```blade
@extends('layouts.app')

@section('title', __('common.manage_files') . ' - Admin')

@section('content')
    <!-- Nội dung trang -->
@endsection
```

**Giải thích:**

#### `@extends('layouts.app')`
- Kế thừa layout chính
- File `layouts/app.blade.php` chứa HTML cơ bản (header, footer, sidebar...)

#### `@section('title', ...)`
- Đặt tiêu đề trang
- `__('common.manage_files')`: Hàm dịch ngôn ngữ (i18n)
- Lấy text từ file `resources/lang/*/common.php`

#### `@section('content')...@endsection`
- Định nghĩa nội dung chính
- Nội dung này sẽ được đặt vào vị trí `@yield('content')` trong layout

---

#### Thẻ Thống Kê

```blade
<div class="col-lg-3 col-md-6 mb-3">
    <div class="card card-block card-stretch card-height">
        <div class="card-body">
            <div class="d-flex align-items-center justify-content-between">
                <div>
                    <h6 class="mb-2 text-muted">{{ __('common.total_files_count') }}</h6>
                    <h3 class="mb-0">{{ number_format($stats['total']) }}</h3>
                </div>
                <div class="icon-small bg-primary-light rounded p-2">
                    <i class="las la-file text-primary"></i>
                </div>
            </div>
        </div>
    </div>
</div>
```

**Giải thích:**
- `{{ $stats['total'] }}`: Hiển thị biến từ Controller
  - `{{ }}`: Blade syntax - Tự động escape HTML (bảo mật)
  - `{!! !!}`: Hiển thị raw HTML (không escape - nguy hiểm!)
- `number_format()`: Format số có dấu phẩy (1000 → 1,000)
- `__('common.total_files_count')`: Dịch ngôn ngữ

**Bootstrap Classes:**
- `col-lg-3`: Chiếm 3/12 cột trên màn lớn (4 card 1 hàng)
- `col-md-6`: Chiếm 6/12 cột trên màn trung (2 card 1 hàng)
- `mb-3`: Margin bottom 3 units
- `d-flex`: Display flex
- `text-muted`: Màu chữ xám nhạt

---

#### Flash Messages (Thông Báo)

```blade
@if(session('status'))
    <div class="alert alert-success alert-dismissible fade show">
        {{ session('status') }}
        <button type="button" class="close" data-dismiss="alert">&times;</button>
    </div>
@endif

@if(session('error'))
    <div class="alert alert-danger alert-dismissible fade show">
        {{ session('error') }}
        <button type="button" class="close" data-dismiss="alert">&times;</button>
    </div>
@endif
```

**Giải thích:**

#### `@if...@endif`
- Điều kiện trong Blade
- Tương đương PHP: `<?php if(...) { ?>`

#### `session('status')`
- Lấy flash message từ session
- Flash message: Thông báo tạm thời, chỉ hiển thị 1 lần
- Set từ Controller: `redirect()->with('status', 'Success!')`

**Luồng Hoạt Động:**
```
1. Controller: redirect()->with('status', 'File deleted!')
2. Laravel lưu vào session tạm
3. View hiển thị: session('status')
4. Sau khi hiển thị → Laravel tự xóa khỏi session
```

---

#### Form Tìm Kiếm & Lọc

```blade
<form method="GET" action="{{ route('admin.files.index') }}" class="row align-items-end">
    <div class="col-md-3 mb-2 mb-md-0">
        <label for="search" class="small text-muted mb-1">{{ __('common.search') }}</label>
        <input type="text" 
               class="form-control" 
               id="search" 
               name="search" 
               placeholder="{{ __('common.file_name_admin') }}..." 
               value="{{ request('search') }}">
    </div>
    <!-- ... các filter khác -->
    <div class="col-md-1">
        <button type="submit" class="btn btn-primary btn-block">
            <i class="las la-search"></i>
        </button>
    </div>
</form>
```

**Giải thích:**

#### `method="GET"`
- Gửi dữ liệu qua URL query string
- Ví dụ: `/admin/files?search=test&user_id=1`
- Thích hợp cho tìm kiếm, lọc (có thể bookmark URL)

#### `action="{{ route('admin.files.index') }}"`
- `route()`: Tạo URL từ tên route
- Lợi ích: Nếu đổi URL trong route, không cần sửa view

#### `value="{{ request('search') }}"`
- `request('search')`: Lấy giá trị từ URL query
- Giữ giá trị đã nhập sau khi submit

**Method GET vs POST:**
- **GET**: Tìm kiếm, lọc, phân trang (URL có tham số)
- **POST**: Thêm, sửa, xóa dữ liệu (URL không có tham số)

---

#### Nút Xóa Bộ Lọc

```blade
@if(request()->hasAny(['search', 'user_id', 'type', 'status', 'sort']))
    <div class="mt-2">
        <a href="{{ route('admin.files.index') }}" class="btn btn-sm btn-outline-secondary">
            <i class="las la-times mr-1"></i> {{ __('common.clear_filters') }}
        </a>
    </div>
@endif
```

**Giải thích:**
- `request()->hasAny([...])`: Kiểm tra có tham số nào đó không
- Chỉ hiển thị nút khi đang có filter
- Click → Quay về URL gốc (không có tham số)

---

#### Bảng Dữ Liệu

```blade
<table class="table mb-0 table-borderless">
    <thead>
        <tr>
            <th style="width: 40px;"><i class="ri-file-line"></i></th>
            <th>{{ __('common.file_name_admin') }}</th>
            <th>{{ __('common.file_owner') }}</th>
            <!-- ... -->
        </tr>
    </thead>
    <tbody>
        @forelse($files as $file)
        <tr>
            <td>
                @php
                    $iconClass = 'ri-file-line';
                    $iconColor = 'text-muted';
                    if(Str::contains($file->mime_type, 'pdf')) {
                        $iconClass = 'ri-file-pdf-line';
                        $iconColor = 'text-danger';
                    } elseif(Str::contains($file->mime_type, 'image')) {
                        $iconClass = 'ri-image-line';
                        $iconColor = 'text-info';
                    }
                    // ...
                @endphp
                <i class="{{ $iconClass }} {{ $iconColor }} font-size-24"></i>
            </td>
            <td>
                <strong>
                    <a href="{{ route('admin.files.show', $file->id) }}" class="text-primary">
                        {{ $file->name }}
                    </a>
                </strong>
            </td>
            <!-- ... -->
        </tr>
        @empty
        <tr>
            <td colspan="8" class="text-center py-4 text-muted">
                <i class="ri-file-line font-size-48 text-muted mb-3 d-block"></i>
                {{ __('common.no_files_found_admin') }}
            </td>
        </tr>
        @endforelse
    </tbody>
</table>
```

**Giải thích:**

#### `@forelse...@empty...@endforelse`
- Vòng lặp với xử lý trường hợp rỗng
- `@forelse($files as $file)`: Lặp qua mỗi file
- `@empty`: Hiển thị khi không có dữ liệu
- Tương đương:
```php
if(count($files) > 0) {
    foreach($files as $file) {
        // Hiển thị file
    }
} else {
    // Hiển thị "No files found"
}
```

#### `@php...@endphp`
- Viết PHP code trong Blade
- **Lưu ý**: Nên hạn chế logic phức tạp trong view
- Logic nên ở Controller hoặc Model

#### `{{ route('admin.files.show', $file->id) }}`
- Tạo URL với tham số
- Ví dụ: `/admin/files/123`

#### `Str::contains($file->mime_type, 'pdf')`
- Helper kiểm tra chuỗi có chứa substring không
- `Str::`: Laravel String helper

---

#### Phân Trang

```blade
@if(method_exists($files, 'links'))
    <div class="card-footer">
        {{ $files->links() }}
    </div>
@endif
```

**Giải thích:**
- `method_exists()`: Kiểm tra object có method không
- `$files->links()`: Hiển thị nút phân trang (1, 2, 3, Next, Previous...)
- Laravel tự động tạo HTML phân trang Bootstrap

**Khi nào có `links()`?**
- Khi dùng `paginate()` trong Controller
- Không có khi dùng `get()` hoặc `all()`

---

#### Modal Xác Nhận Xóa

```blade
@foreach($files as $file)
<div class="modal fade" id="deleteFileModal{{ $file->id }}" tabindex="-1" role="dialog">
    <div class="modal-dialog" role="document">
        <div class="modal-content">
            <div class="modal-header bg-danger text-white">
                <h5 class="modal-title text-white">{{ __('common.delete_file') }}</h5>
                <button type="button" class="close text-white" data-dismiss="modal">
                    <span aria-hidden="true">&times;</span>
                </button>
            </div>
            <form action="{{ route('admin.files.destroy', $file) }}" method="POST">
                @csrf
                @method('DELETE')
                <div class="modal-body">
                    <div class="alert alert-danger">
                        <i class="ri-error-warning-line mr-2"></i>
                        <strong>{{ __('common.warning') }}:</strong> 
                        {{ __('common.warning_action_cannot_undo') }}
                    </div>
                    <p>{{ __('common.delete_file_confirm') }}</p>
                    <p class="mb-2"><strong>{{ $file->name }}</strong></p>
                </div>
                <div class="modal-footer">
                    <button type="button" class="btn btn-secondary" data-dismiss="modal">
                        {{ __('common.cancel') }}
                    </button>
                    <button type="submit" class="btn btn-danger">
                        {{ __('common.delete_file') }}
                    </button>
                </div>
            </form>
        </div>
    </div>
</div>
@endforeach
```

**Giải thích:**

#### Modal ID Động
```blade
id="deleteFileModal{{ $file->id }}"
```
- Mỗi file có modal riêng
- Ví dụ: `deleteFileModal1`, `deleteFileModal2`...

#### `@csrf`
- **CSRF Token**: Bảo vệ khỏi Cross-Site Request Forgery
- Laravel yêu cầu token cho mọi POST request
- Tự động tạo input hidden: `<input type="hidden" name="_token" value="...">`

#### `@method('DELETE')`
- HTML form chỉ hỗ trợ GET và POST
- Laravel dùng method spoofing để fake DELETE
- Tạo input hidden: `<input type="hidden" name="_method" value="DELETE">`

#### Trigger Modal
```blade
<button type="button" 
        class="btn btn-sm btn-outline-danger" 
        data-toggle="modal" 
        data-target="#deleteFileModal{{ $file->id }}">
    <i class="las la-trash"></i>
</button>
```
- `data-toggle="modal"`: Kích hoạt modal
- `data-target="#deleteFileModal1"`: Mở modal có ID tương ứng

---

### 2. `show.blade.php` - Chi Tiết File

#### Breadcrumb (Đường Dẫn)

```blade
<nav aria-label="breadcrumb">
    <ol class="breadcrumb bg-white">
        @if(request('from') === 'favorites')
            <li class="breadcrumb-item">
                <a href="{{ route('admin.favorites.index') }}">
                    {{ __('common.manage_favorites') }}
                </a>
            </li>
        @else
            <li class="breadcrumb-item">
                <a href="{{ route('admin.files.index') }}">
                    {{ __('common.manage_files') }}
                </a>
            </li>
            @if($file->folder)
                <li class="breadcrumb-item">
                    <a href="{{ route('admin.folders.show', $file->folder->id) }}">
                        {{ $file->folder->name }}
                    </a>
                </li>
            @endif
        @endif
        <li class="breadcrumb-item active" aria-current="page">
            {{ $file->name }}
        </li>
    </ol>
</nav>
```

**Giải thích:**
- Breadcrumb: Hiển thị vị trí hiện tại
- `request('from')`: Lấy tham số từ URL
- Ví dụ: Home > Files > Documents > file.pdf
- Giúp người dùng biết đang ở đâu và quay lại dễ dàng

**Ví dụ URL:**
```
/admin/files/123?from=favorites
→ Favorites > file.pdf

/admin/files/123
→ Files > Documents > file.pdf
```

---

#### Icon Động Theo Loại File

```blade
@php
    $iconClass = 'ri-file-line';
    $iconColor = 'text-muted';
    
    if(Str::contains($file->mime_type, 'pdf')) {
        $iconClass = 'ri-file-pdf-line';
        $iconColor = 'text-danger';
    } elseif(Str::contains($file->mime_type, 'word')) {
        $iconClass = 'ri-file-word-line';
        $iconColor = 'text-primary';
    } elseif(Str::contains($file->mime_type, 'image')) {
        $iconClass = 'ri-image-line';
        $iconColor = 'text-info';
    }
    // ...
@endphp

<i class="{{ $iconClass }} {{ $iconColor }} font-size-32 mr-3"></i>
```

**Giải thích:**
- Hiển thị icon phù hợp với loại file
- PDF → Icon màu đỏ
- Word → Icon màu xanh
- Image → Icon màu xanh nhạt

---

#### Hiển Thị Avatar User

```blade
@if($file->user->avatar)
    <img src="{{ $file->user->avatar_url }}" 
         alt="{{ $file->user->name }}" 
         class="rounded-circle mr-2" 
         style="width: 40px; height: 40px; object-fit: cover;"
         onerror="this.src='{{ asset('assets/images/user/1.jpg') }}'">
@else
    <div class="rounded-circle bg-primary text-white d-inline-flex align-items-center justify-content-center mr-2" 
         style="width: 40px; height: 40px; font-size: 16px;">
        {{ strtoupper(substr($file->user->name, 0, 1)) }}
    </div>
@endif
```

**Giải thích:**

#### `@if...@else...@endif`
- Điều kiện có nhánh else
- Nếu có avatar → Hiển thị ảnh
- Nếu không → Hiển thị chữ cái đầu

#### `onerror="..."`
- JavaScript xử lý khi ảnh lỗi
- Chuyển sang ảnh mặc định

#### `{{ strtoupper(substr($file->user->name, 0, 1)) }}`
- `substr(..., 0, 1)`: Lấy ký tự đầu tiên
- `strtoupper()`: Chuyển thành chữ hoa
- Ví dụ: "John" → "J"

#### `asset('assets/images/user/1.jpg')`
- Tạo URL đến file trong thư mục `public/`
- `/assets/images/user/1.jpg`

---

### 3. `view.blade.php` - Xem Trước File

#### Phát Hiện Loại File

```blade
@php
    $mime = strtolower($file->mime_type ?? '');
    $ext = strtolower($file->extension ?? '');
    
    $isImage = Str::startsWith($mime, 'image/');
    $isPdf = Str::contains($mime, 'pdf');
    $isVideo = Str::startsWith($mime, 'video/');
    $isAudio = Str::startsWith($mime, 'audio/');
    $isText = Str::startsWith($mime, 'text/') || in_array($ext, ['txt','md','json']);
    $isDocx = Str::contains($mime, 'officedocument.wordprocessingml') || $ext === 'docx';
    $isExcel = Str::contains($mime, 'spreadsheetml') || in_array($ext, ['xlsx', 'xls']);
@endphp
```

**Giải thích:**
- Phát hiện loại file để hiển thị phù hợp
- `??`: Null coalescing operator - Nếu null thì dùng giá trị mặc định
- `Str::startsWith()`: Kiểm tra bắt đầu bằng...
- `in_array()`: Kiểm tra có trong mảng không

---

#### Hiển Thị Ảnh

```blade
@if($isImage)
    <img src="{{ $fileUrl }}" 
         alt="{{ $file->original_name }}" 
         class="img-fluid" 
         style="max-height: 75vh;">
@endif
```

**Giải thích:**
- `img-fluid`: Bootstrap class - Ảnh responsive
- `max-height: 75vh`: Tối đa 75% chiều cao viewport
- `vh`: Viewport Height (đơn vị CSS)

---

#### Hiển Thị PDF

```blade
@elseif($isPdf)
    <iframe src="{{ $fileUrl }}#toolbar=1" 
            style="width:100%; height:75vh; border: none;" 
            title="PDF preview">
    </iframe>
@endif
```

**Giải thích:**
- `iframe`: Nhúng nội dung từ URL khác
- `#toolbar=1`: Hiển thị toolbar của PDF viewer
- Trình duyệt tự động render PDF

---

#### Hiển Thị Video/Audio

```blade
@elseif($isVideo)
    <video controls style="width:100%; max-height:75vh; background:#000;">
        <source src="{{ $fileUrl }}" type="{{ $file->mime_type }}" />
        Your browser does not support the video tag.
    </video>
@elseif($isAudio)
    <audio controls style="width:100%;">
        <source src="{{ $fileUrl }}" type="{{ $file->mime_type }}" />
        Your browser does not support the audio element.
    </audio>
@endif
```

**Giải thích:**
- `controls`: Hiển thị nút play, pause, volume...
- `<source>`: Chỉ định nguồn file
- Text trong tag: Hiển thị khi trình duyệt không hỗ trợ

---

#### Hiển Thị File Word (DOCX)

```blade
@elseif($isDocx)
    <div id="docxContainer" class="w-100" style="max-width: 900px; margin: 0 auto;">
        <div id="docxLoading" class="text-center text-muted">
            <i class="ri-loader-4-line ri-spin"></i> {{ __('common.loading_document') }}...
        </div>
    </div>
    
    @push('scripts')
    <script src="https://cdn.jsdelivr.net/npm/mammoth@1.6.0/mammoth.browser.min.js"></script>
    <script>
        (function(){
            const url = @json($fileUrl);
            const container = document.getElementById('docxContainer');
            const loading = document.getElementById('docxLoading');
            
            fetch(url).then(r => {
                if(!r.ok) throw new Error('Network error');
                return r.arrayBuffer();
            }).then(arrayBuffer => {
                return window.mammoth.convertToHtml({arrayBuffer});
            }).then(result => {
                loading && loading.remove();
                const wrapper = document.createElement('div');
                wrapper.className = 'docx-content';
                wrapper.innerHTML = result.value;
                container.appendChild(wrapper);
            }).catch(err => {
                loading && (loading.innerHTML = '{{ __('common.preview_not_available') }}');
            });
        })();
    </script>
    @endpush
@endif
```

**Giải thích:**

#### `@push('scripts')...@endpush`
- Đẩy script vào stack
- Sẽ được đặt ở vị trí `@stack('scripts')` trong layout
- Thường ở cuối body để tối ưu tốc độ

#### `@json($fileUrl)`
- Chuyển biến PHP thành JSON
- Blade directive an toàn hơn `json_encode()`

#### Mammoth.js
- Thư viện JavaScript chuyển DOCX sang HTML
- `fetch()`: API lấy file
- `arrayBuffer()`: Đọc file dạng binary
- `mammoth.convertToHtml()`: Chuyển đổi

**Luồng Hoạt Động:**
```
1. Hiển thị loading spinner
2. Fetch file DOCX từ server
3. Đọc file dưới dạng ArrayBuffer
4. Dùng Mammoth chuyển sang HTML
5. Xóa loading, hiển thị nội dung
6. Nếu lỗi → Hiển thị thông báo
```

---

#### Hiển Thị File Excel

```blade
@elseif($isExcel)
    <div id="excelContainer" class="w-100">
        <div id="excelLoading" class="text-center text-muted py-5">
            <i class="ri-loader-4-line ri-spin font-size-32"></i>
            <p class="mt-3">{{ __('common.loading') }}...</p>
        </div>
        <div id="excelContent" style="display: none;">
            <div class="mb-3">
                <select id="excelSheetSelect" class="form-control" style="max-width: 300px;">
                    <option value="">{{ __('common.select_sheet') }}</option>
                </select>
            </div>
            <div id="excelTableWrapper" style="overflow-x: auto; max-height: 70vh;">
                <table id="excelTable" class="table table-bordered table-sm mb-0">
                    <thead id="excelTableHead"></thead>
                    <tbody id="excelTableBody"></tbody>
                </table>
            </div>
        </div>
    </div>
    
    @push('scripts')
    <script src="https://cdnjs.cloudflare.com/ajax/libs/xlsx/0.18.5/xlsx.full.min.js"></script>
    <script>
        (function(){
            const url = @json($fileUrl);
            let workbook = null;
            
            fetch(url).then(r => {
                if(!r.ok) throw new Error('Network error');
                return r.arrayBuffer();
            }).then(arrayBuffer => {
                workbook = XLSX.read(arrayBuffer, {type: 'array'});
                
                // Populate sheet selector
                workbook.SheetNames.forEach((name, index) => {
                    const option = document.createElement('option');
                    option.value = index;
                    option.textContent = name;
                    if(index === 0) option.selected = true;
                    sheetSelect.appendChild(option);
                });
                
                // Load first sheet
                loadSheet(0);
                
                loading.style.display = 'none';
                content.style.display = 'block';
            }).catch(err => {
                loading.innerHTML = '<div class="text-danger">Unable to load file</div>';
            });
            
            function loadSheet(sheetIndex) {
                const sheetName = workbook.SheetNames[sheetIndex];
                const worksheet = workbook.Sheets[sheetName];
                
                // Convert to JSON
                const jsonData = XLSX.utils.sheet_to_json(worksheet, {header: 1, defval: ''});
                
                // Build table...
            }
        })();
    </script>
    @endpush
@endif
```

**Giải thích:**

#### SheetJS (XLSX)
- Thư viện đọc/ghi Excel
- `XLSX.read()`: Đọc file Excel
- `sheet_to_json()`: Chuyển sheet thành JSON

#### Multiple Sheets
- Excel có thể có nhiều sheet
- Tạo dropdown để chọn sheet
- `loadSheet()`: Load sheet được chọn

---

## 📁 AdminFoldersController Views

### 1. `index.blade.php` - Danh Sách Folders

Tương tự Files index nhưng có thêm:

#### Hiển thị Folder Color

```blade
<i class="ri-folder-3-fill font-size-24" 
   style="color: {{ $folder->color ?? '#3498db' }}">
</i>
```

**Giải thích:**
- Mỗi folder có thể có màu riêng
- `??`: Nếu không có màu → Dùng màu mặc định xanh

#### Hiển thị Số Lượng Con

```blade
@if($folder->children_count > 0)
    <small class="text-info">
        <i class="ri-folder-line"></i> 
        {{ $folder->children_count }} {{ __('common.folder_children_count') }}
    </small>
@endif
```

**Giải thích:**
- `children_count`: Từ `withCount('children')` trong Controller
- Chỉ hiển thị khi có folder con

---

### 2. `show.blade.php` - Chi Tiết Folder

#### Hiển Thị Folder Parent/Children

```blade
<div class="row">
    <div class="col-md-6">
        <h6 class="text-muted">{{ __('common.folder_parent') }}</h6>
        @if($folder->parent)
            <a href="{{ route('admin.folders.show', $folder->parent->id) }}" 
               class="btn btn-sm btn-outline-secondary">
                <i class="ri-folder-line"></i> {{ $folder->parent->name }}
            </a>
        @else
            <span class="text-muted">{{ __('common.folder_no_parent') }}</span>
        @endif
    </div>
</div>
```

**Giải thích:**
- Hiển thị folder cha (nếu có)
- Link đến folder cha để dễ navigate

---

#### Danh Sách Subfolders & Files

```blade
@if($folder->children->count() > 0)
    <h5 class="mt-4 mb-3">
        <i class="ri-folder-line mr-2"></i>
        {{ __('common.folder_subfolders') }} ({{ $folder->children->count() }})
    </h5>
    <div class="row">
        @foreach($folder->children as $child)
        <div class="col-md-4 mb-3">
            <div class="card">
                <div class="card-body">
                    <div class="d-flex align-items-center">
                        <i class="ri-folder-3-fill font-size-32 mr-3" 
                           style="color: {{ $child->color ?? '#3498db' }}">
                        </i>
                        <div class="flex-grow-1">
                            <h6 class="mb-1">
                                <a href="{{ route('admin.folders.show', $child->id) }}">
                                    {{ $child->name }}
                                </a>
                            </h6>
                            <small class="text-muted">
                                {{ $child->files_count }} {{ __('common.files') }}
                            </small>
                        </div>
                    </div>
                </div>
            </div>
        </div>
        @endforeach
    </div>
@endif
```

**Giải thích:**
- Hiển thị grid các subfolder
- `col-md-4`: 3 folder trên 1 hàng
- Link đến chi tiết folder con

---

## ⭐ AdminFavoritesController Views

### `index.blade.php` - Danh Sách Favorites

#### Gộp Files & Folders

```blade
@forelse($paginator as $item)
<tr>
    <td>
        @if($item['type'] === 'folder')
            <i class="ri-folder-3-fill font-size-24" 
               style="color: {{ $item['model']->color ?? '#3498db' }}">
            </i>
        @else
            @php
                $iconClass = 'ri-file-line';
                if(Str::contains($item['mime_type'], 'pdf')) {
                    $iconClass = 'ri-file-pdf-line';
                }
                // ...
            @endphp
            <i class="{{ $iconClass }} font-size-24"></i>
        @endif
    </td>
    <td>
        @if($item['type'] === 'folder')
            <a href="{{ route('admin.folders.show', ['folder' => $item['id'], 'from' => 'favorites']) }}">
                {{ $item['name'] }}
            </a>
        @else
            <a href="{{ route('admin.files.show', ['file' => $item['id'], 'from' => 'favorites']) }}">
                {{ $item['name'] }}
            </a>
        @endif
    </td>
    <!-- ... -->
</tr>
@empty
<tr>
    <td colspan="7" class="text-center py-4 text-muted">
        <i class="ri-star-line font-size-48 mb-3 d-block"></i>
        {{ __('common.no_favorites_found') }}
    </td>
</tr>
@endforelse
```

**Giải thích:**
- Kiểm tra `type` để hiển thị phù hợp
- Folder → Icon folder + Link đến folder
- File → Icon file + Link đến file
- `from=favorites`: Tham số để breadcrumb biết đến từ favorites

---

## 🎯 Các Blade Directives Quan Trọng

### 1. Hiển Thị Dữ Liệu

```blade
{{ $variable }}              <!-- Escape HTML (an toàn) -->
{!! $variable !!}            <!-- Raw HTML (nguy hiểm) -->
{{ $var ?? 'default' }}      <!-- Null coalescing -->
{{ $var ?: 'default' }}      <!-- Ternary shorthand -->
```

### 2. Điều Kiện

```blade
@if($condition)
    <!-- Code -->
@elseif($otherCondition)
    <!-- Code -->
@else
    <!-- Code -->
@endif

@unless($condition)          <!-- if NOT -->
    <!-- Code -->
@endunless

@isset($variable)            <!-- if isset() -->
    <!-- Code -->
@endisset

@empty($variable)            <!-- if empty() -->
    <!-- Code -->
@endempty
```

### 3. Vòng Lặp

```blade
@foreach($items as $item)
    <!-- Code -->
@endforeach

@forelse($items as $item)
    <!-- Code -->
@empty
    <!-- Hiển thị khi rỗng -->
@endforelse

@for($i = 0; $i < 10; $i++)
    <!-- Code -->
@endfor

@while($condition)
    <!-- Code -->
@endwhile
```

### 4. Loop Variables

```blade
@foreach($items as $item)
    {{ $loop->index }}       <!-- 0, 1, 2, ... -->
    {{ $loop->iteration }}   <!-- 1, 2, 3, ... -->
    {{ $loop->first }}       <!-- true ở lần đầu -->
    {{ $loop->last }}        <!-- true ở lần cuối -->
    {{ $loop->count }}       <!-- Tổng số item -->
    {{ $loop->remaining }}   <!-- Số item còn lại -->
@endforeach
```

### 5. Include & Components

```blade
@include('partials.header')                     <!-- Include view -->
@include('partials.item', ['item' => $item])    <!-- Truyền data -->

<x-alert type="success">Message</x-alert>       <!-- Component -->
```

### 6. Authentication

```blade
@auth
    <!-- User đã đăng nhập -->
@endauth

@guest
    <!-- User chưa đăng nhập -->
@endguest

@auth('admin')
    <!-- User đăng nhập với guard 'admin' -->
@endauth
```

### 7. Section & Yield

```blade
<!-- Layout: layouts/app.blade.php -->
<!DOCTYPE html>
<html>
<head>
    <title>@yield('title', 'Default Title')</title>
</head>
<body>
    @yield('content')
    @stack('scripts')
</body>
</html>

<!-- View: pages/home.blade.php -->
@extends('layouts.app')

@section('title', 'Home Page')

@section('content')
    <p>Content here</p>
@endsection

@push('scripts')
    <script>alert('Hello');</script>
@endpush
```

---

## 🎨 Bootstrap Classes Thường Dùng

### Layout

```html
<div class="container">         <!-- Fixed width container -->
<div class="container-fluid">   <!-- Full width container -->
<div class="row">               <!-- Row (flex container) -->
<div class="col-12">            <!-- Column chiếm 12/12 -->
<div class="col-md-6">          <!-- Column chiếm 6/12 trên màn md+ -->
```

### Spacing

```html
<div class="m-3">    <!-- Margin all sides: 3 units -->
<div class="mt-2">   <!-- Margin top: 2 units -->
<div class="mb-4">   <!-- Margin bottom: 4 units -->
<div class="p-3">    <!-- Padding all sides: 3 units -->
<div class="pt-2">   <!-- Padding top: 2 units -->
```

**Units:** 0, 1, 2, 3, 4, 5 (0 = 0, 1 = 0.25rem, 2 = 0.5rem, ...)

### Display & Flex

```html
<div class="d-flex">                    <!-- display: flex -->
<div class="d-none">                    <!-- display: none -->
<div class="d-block">                   <!-- display: block -->
<div class="justify-content-between">   <!-- space-between -->
<div class="align-items-center">        <!-- align-items: center -->
```

### Text

```html
<p class="text-center">     <!-- text-align: center -->
<p class="text-left">       <!-- text-align: left -->
<p class="text-right">      <!-- text-align: right -->
<p class="text-muted">      <!-- màu xám nhạt -->
<p class="text-primary">    <!-- màu chính -->
<p class="text-danger">     <!-- màu đỏ -->
<p class="font-weight-bold"><!-- chữ đậm -->
```

### Components

```html
<button class="btn btn-primary">Button</button>
<span class="badge badge-success">Success</span>
<div class="alert alert-danger">Error message</div>
<div class="card">
    <div class="card-header">Header</div>
    <div class="card-body">Body</div>
    <div class="card-footer">Footer</div>
</div>
```

---

## 🔐 Security Best Practices Trong Views

### 1. Luôn Escape Output

```blade
<!-- ✅ TỐT - Tự động escape -->
{{ $user->name }}

<!-- ❌ XẤU - Không escape (XSS risk) -->
{!! $user->name !!}

<!-- ✅ TỐT - Dùng {!! !!} chỉ khi cần HTML -->
{!! $trustedHtmlContent !!}
```

### 2. Validate URL

```blade
<!-- ✅ TỐT - Dùng route() -->
<a href="{{ route('admin.files.show', $file->id) }}">View</a>

<!-- ❌ XẤU - Hard-code URL -->
<a href="/admin/files/{{ $file->id }}">View</a>
```

### 3. CSRF Protection

```blade
<!-- ✅ TỐT - Có @csrf -->
<form method="POST" action="{{ route('files.store') }}">
    @csrf
    <!-- Form fields -->
</form>

<!-- ❌ XẤU - Thiếu @csrf -->
<form method="POST" action="{{ route('files.store') }}">
    <!-- Form fields - SẼ BỊ LỖI! -->
</form>
```

### 4. Authorize Actions

```blade
<!-- ✅ TỐT - Kiểm tra quyền -->
@can('delete', $file)
    <button onclick="confirmDelete()">Delete</button>
@endcan

<!-- ❌ XẤU - Không kiểm tra quyền -->
<button onclick="confirmDelete()">Delete</button>
```

---

## 💡 Tips & Tricks

### 1. Debug Trong Blade

```blade
@dump($variable)       <!-- Dump variable -->
@dd($variable)         <!-- Dump and die -->

{{ dd($files) }}       <!-- Debug collection -->
```

### 2. Inline If-Else

```blade
<!-- Cách 1: Ternary -->
{{ $file->is_trash ? 'Trash' : 'Active' }}

<!-- Cách 2: Null coalescing -->
{{ $file->description ?? 'No description' }}

<!-- Cách 3: @if inline -->
<span class="@if($file->is_favorite) text-warning @endif">Star</span>
```

### 3. Formatting Numbers & Dates

```blade
<!-- Number formatting -->
{{ number_format($file->size / 1024 / 1024, 2) }} MB

<!-- Date formatting -->
{{ $file->created_at->format('d/m/Y H:i') }}
{{ $file->created_at->diffForHumans() }}  <!-- 2 hours ago -->

<!-- Timezone -->
{{ $file->created_at->timezone('Asia/Ho_Chi_Minh')->format('d/m/Y') }}
```

### 4. String Helpers

```blade
{{ Str::limit($text, 50) }}              <!-- Cắt chuỗi -->
{{ Str::slug($name) }}                   <!-- URL-friendly -->
{{ Str::upper($text) }}                  <!-- Chữ hoa -->
{{ Str::lower($text) }}                  <!-- Chữ thường -->
{{ Str::contains($mime, 'pdf') }}        <!-- Có chứa? -->
{{ Str::startsWith($mime, 'image/') }}   <!-- Bắt đầu bằng? -->
```

### 5. Asset Helpers

```blade
{{ asset('css/app.css') }}                    <!-- /css/app.css -->
{{ url('profile') }}                          <!-- http://domain.com/profile -->
{{ route('admin.files.index') }}              <!-- Named route URL -->
{{ route('admin.files.show', $file->id) }}    <!-- Route with parameter -->
```

---

## 📚 Tổng Kết

### Luồng MVC Hoàn Chỉnh

```
1. USER REQUEST
   ↓
   http://domain.com/admin/files?search=test

2. ROUTE (web.php)
   ↓
   Route::get('/admin/files', [AdminFilesController::class, 'index']);

3. CONTROLLER
   ↓
   public function index(Request $request) {
       $files = File::where('name', 'like', "%{$request->search}%")
                    ->paginate(20);
       return view('pages.admin.files.index', compact('files'));
   }

4. MODEL
   ↓
   SELECT * FROM files WHERE name LIKE '%test%' LIMIT 20;

5. VIEW (index.blade.php)
   ↓
   @foreach($files as $file)
       <tr>
           <td>{{ $file->name }}</td>
       </tr>
   @endforeach

6. RESPONSE
   ↓
   HTML được trả về trình duyệt
```

### Các File Quan Trọng

```
app/
  Http/
    Controllers/
      Admin/
        AdminFilesController.php      ← Logic xử lý
        AdminFoldersController.php
        AdminFavoritesController.php
  Models/
    File.php                          ← Tương tác database
    Folder.php

resources/
  views/
    layouts/
      app.blade.php                   ← Layout chính
    pages/
      admin/
        files/
          index.blade.php             ← View danh sách
          show.blade.php              ← View chi tiết
          view.blade.php              ← View xem trước
        folders/
          index.blade.php
          show.blade.php
        favorites/
          index.blade.php

routes/
  web.php                             ← Định nghĩa routes
```

---

**Chúc bạn học tốt! 🎉**

Nếu có thắc mắc, hãy đọc lại từng phần nhỏ và thử nghiệm trên code thực tế.
