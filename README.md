## Sale Order

ที่มา: `addons/tifshop_sale_sync`

- Get orders จาก Zort ใช้ `Order/GetOrders` ผ่าน `_fetch_zort_orders()`
- Create API ใช้ `/api/v1/sale/order`
- Update API ใช้ `/api/v1/sale/order/update`
- Delete API ใช้ `/api/v1/sale/order/delete`
- Create/Update ใช้ payload schema จาก `SaleOrderDTO`
- Logic สร้าง/อัปเดต SO อยู่ที่ `res.config.settings.process_single_order()`

### Flow หลัก

| Step | Odoo | Zort | Remake |
| ----- | ---- | ---- | ------ |
| Get orders | `_fetch_zort_orders()` | `Order/GetOrders` | ดึง list order จาก Zort โดยส่ง `page`, `limit`, `createdafter` |
| Check existing SO | `sale.order.zort_id` | `id` | ใช้ `id` จาก Zort หา SO เดิมใน Odoo |
| Create SO | `sale.order.create(base_vals)` | payload order | ถ้ายังไม่มี `zort_id` ใน Odoo จะสร้าง SO ใหม่ |
| Update SO | `sale.order.write(base_vals)` | payload order | ถ้ามี `zort_id` อยู่แล้ว จะ update SO และ sync line |
| Delete SO | `sale.order.search([("zort_id", "=", id)])` | `id` | หา SO จาก `id` ของ Zort แล้ว cancel + unlink |

### Sale Order Fields

| Fields | Odoo | Zort | Remake |
| ----- | ---- | ---- | ------ |
| zort_id | `sale.order.zort_id` | `id` | เก็บ id จาก Zort ลง field `zort_id` ใน Odoo และใช้เป็น key หา order เดิม |
| partner_id | `sale.order.partner_id` | `customerid`, `customername`, `customercode` | หา/สร้าง partner จากข้อมูลลูกค้า แล้วเอา partner id มาใส่ SO |
| zort_name | `sale.order.zort_name` | `number` | เก็บเลข order จาก Zort ไว้ใน `zort_name` |
| date_order | `sale.order.date_order` | `orderdate` | parse วันที่จาก Zort เป็น `%Y-%m-%d %H:%M:%S` |
| amount_total | `sale.order.amount_total` | `amount` | ใส่ยอดรวมจาก Zort ลง `amount_total` โดย cast เป็น float |
| zort_payment_state | `sale.order.zort_payment_state` | `paymentstatus` | map สถานะจ่ายเงินจาก Zort เป็น selection ใน Odoo |
| sale_channel | `sale.order.sale_channel` | `saleschannel` | ใช้ช่องทางขายจาก Zort ถ้าไม่มีใช้ `"Sale Order"` |
| discount | `sale.order.discount` | `discount` | parse discount จาก Zort ถ้าว่าง/อ่านไม่ได้ใช้ `0.0` |
| state / picking status | `sale.order.state`, `stock.picking.state` | `status` | ไม่เขียน `state` ตรงใน base vals แต่ใช้ `status` เพื่อ confirm/cancel/validate/return picking |

### Sale Order Line Fields

| Fields | Odoo | Zort | Remake |
| ----- | ---- | ---- | ------ |
| zort_id | `sale.order.line.zort_id` | `list[].id` | เก็บ id ของรายการสินค้าใน Zort ลง line และใช้หา line เดิมตอน update |
| product_id | `sale.order.line.product_id` | `list[].productid` | หา product จาก `product.template.zort_id`; ถ้าไม่เจอจะสร้าง product ใหม่จากข้อมูล item |
| name | `sale.order.line.name` | `list[].name` | ใช้ชื่อ item จาก Zort เป็น description ของ line |
| product_uom_qty | `sale.order.line.product_uom_qty` | `list[].number` | จำนวนสินค้า แปลงเป็น float |
| price_unit | `sale.order.line.price_unit` | `list[].pricepernumber_pretax` | ใช้ราคาก่อน VAT ต่อหน่วยจาก Zort |
| discount | `sale.order.line.discount` | `list[].discount`, `list[].pricepernumber` | ถ้าเป็น line จะคำนวณเปอร์เซ็นต์จาก discount amount เทียบกับ `pricepernumber`; ถ้าไม่มีใช้ `0.0` |

### Partner Mapping ตอนหา/สร้างลูกค้า

| Fields | Odoo | Zort | Remake |
| ----- | ---- | ---- | ------ |
| partner lookup | `res.partner.zort_id` | `customerid` | หา partner active ที่ `zort_id = customerid` ก่อน |
| partner lookup fallback | `res.partner.name`, `res.partner.ref` | `customername`, `customercode` | ถ้าหาด้วย zort_id ไม่เจอ จะหาโดยชื่อและรหัสลูกค้า |
| zort_id | `res.partner.zort_id` | `Contact/GetContactDetail.id` | ถ้ามี `customerid` จะเรียก `Contact/GetContactDetail` แล้วสร้าง partner ด้วย id จาก contact detail |
| ref | `res.partner.ref` | `Contact/GetContactDetail.code` | เก็บ code ลูกค้า Zort |
| name | `res.partner.name` | `Contact/GetContactDetail.name` หรือ `customername` | ถ้า fetch contact สำเร็จใช้ detail จาก Zort; ถ้า fail ใช้ข้อมูลใน order |
| phone | `res.partner.phone` | `Contact/GetContactDetail.phone` หรือ `customerphone` | ใช้เบอร์จาก contact detail หรือ fallback จาก order |
| email | `res.partner.email` | `Contact/GetContactDetail.email` หรือ `customeremail` | ใช้อีเมลจาก contact detail หรือ fallback จาก order |
| street | `res.partner.street` | `Contact/GetContactDetail.address` หรือ `customeraddress` | ใช้ที่อยู่จาก contact detail หรือ fallback จาก order |
| vat | `res.partner.vat` | `Contact/GetContactDetail.idnumber` | เก็บเลขผู้เสียภาษีเมื่อตอนสร้างจาก contact detail |
| branch_name | `res.partner.branch_name` | `Contact/GetContactDetail.branchname` | เก็บชื่อสาขาเมื่อตอนสร้างจาก contact detail |
| branch_no | `res.partner.branch_no` | `Contact/GetContactDetail.branchno` | เก็บเลขสาขาเมื่อตอนสร้างจาก contact detail |
| social fields | `facebook`, `line`, `instagram` | `facebook`, `line`, `instagram` | เก็บข้อมูล social เมื่อตอนสร้างจาก contact detail |
| shipping fields | `shipping_name`, `shipping_phone`, `shipping_address` | `shippingName`, `shippingPhone`, `shippingAddress` | เก็บข้อมูลจัดส่งของ partner เมื่อตอนสร้างจาก contact detail |

### Product Mapping ตอนหา/สร้างสินค้าใน Order Line

| Fields | Odoo | Zort | Remake |
| ----- | ---- | ---- | ------ |
| product lookup | `product.template.zort_id` | `list[].productid` | หา product template จาก Zort product id |
| zort_id | `product.template.zort_id` | `list[].productid` | ถ้าไม่เจอสินค้า จะสร้าง product โดยเอา `productid` ไปใส่เป็น `id` เพื่อ prepare เป็น `zort_id` |
| default_code | `product.template.default_code` | `list[].sku` | SKU จาก Zort |
| name | `product.template.name` | `list[].name` | ชื่อสินค้าจาก item |
| type / stock | `product.template.type`, `is_storable` | - | สร้างเป็น consumable และ storable (`type = "consu"`, `is_storable = True`) |

### Order Status Mapping

| Fields | Odoo | Zort | Remake |
| ----- | ---- | ---- | ------ |
| status | `draft` | `Pending` | map เป็น draft |
| status | `sale` | `Success` | map เป็น sale และ validate outgoing picking |
| status | `cancel` | `Voided` | map เป็น cancel และ cancel order/picking |
| status | `waiting` | `Waiting` | map เป็น waiting |
| status | `done` | `Shipping` | map เป็น done และ validate outgoing picking |
| status | `return` | `Returned` | map เป็น return และทำ return picking |
| status | `cancel` | `Failed Shipment` | map เป็น cancel |
| status | `packed` | `Packed` | map เป็น packed และเรียก `action_mark_as_packed()` |

### Payment Status Mapping

| Fields | Odoo | Zort | Remake |
| ----- | ---- | ---- | ------ |
| zort_payment_state | `pending` | `Pending` | map เป็น pending |
| zort_payment_state | `paid` | `Paid` | map เป็น paid |
| zort_payment_state | `voided` | `Voided` | map เป็น voided |
| zort_payment_state | `partial_payment` | `Partial Payment` | map เป็น partial_payment |
| zort_payment_state | `excess_payment` | `Excess Payment` | map เป็น excess_payment |

### Field ที่รับเข้ามาใน DTO แต่ยังไม่ถูกนำไปสร้าง SO โดยตรง

| Fields | Odoo | Zort | Remake |
| ----- | ---- | ---- | ------ |
| shipping info | - | `shippingchannel`, `shippingamount`, `shippingdate`, `shippingname`, `shippingaddress`, `shippingphone`, `shippingemail`, `trackingno`, `trackingList` | DTO รับไว้ แต่ `_prepare_order_vals()` ไม่ map ลง SO โดยตรงใน addon นี้ |
| payment list | - | `payments` | DTO รับไว้ แต่ไม่ได้สร้าง payment/account move ใน flow นี้ |
| vat/tax | - | `vattype`, `vatpercent`, `vatamount`, `*_vat`, `*_pretax` | มี helper `_get_or_create_vat()` แต่ order line ปัจจุบันยังไม่ได้ใส่ `tax_id` จาก helper นี้ |
| warehouse | - | `warehousecode` | DTO รับไว้ แต่ Sale Order sync flow นี้ยังไม่ map warehouse ลง SO/picking โดยตรง |
| tags | - | `tag` | DTO รับไว้ แต่ยังไม่ map ลง Odoo tag |
| description/reference | - | `description`, `reference` | DTO รับไว้ แต่ `_prepare_order_vals()` ยังไม่ map ลง SO field |

## Product Sync

ที่มา: `addons/tifshop_product_sync`

- Batch sync ใช้ `Product/GetProducts` ผ่าน `res.config.settings.sync_products()`
- Create API ใช้ `/api/v1/product/product`
- Update API ใช้ `/api/v1/product/update`
- Delete API ใช้ `/api/v1/product/delete`
- Create/Update API ใช้ payload schema จาก `ProductCreateDTO`
- Delete API ใช้ payload schema จาก `ProductDeleteDTO`
- Logic prepare field หลักอยู่ที่ `product.template._prepare_product_variant()`

### Flow หลัก

| Step | Odoo | Zort | Remake |
| ----- | ---- | ---- | ------ |
| Get products | `_fetch_products()` | `Product/GetProducts` | ดึงสินค้าแยกตาม warehouse โดยส่ง `warehousecode`, `page`, `limit` |
| Check response | `_check_zort_response()` | `res.resCode` | ต้องเป็น `200` ถ้าไม่ใช่จะ raise error |
| Check existing product | `product.template.zort_id` | `id` | ใช้ id จาก Zort หา product เดิมใน Odoo |
| Create product | `product.template.create(vals)` | payload product | ถ้าไม่เจอ `zort_id` จะสร้าง product ใหม่ |
| Update product | `product.template.write(vals)` | payload product | ถ้าเจอ `zort_id` จะ update product |
| Apply stock on create | `stock.quant.inventory_quantity` | `stock` | ตอนสร้างสินค้าใหม่ ถ้า stock มากกว่า 0 จะสร้าง quant แล้ว apply inventory |
| Update stock | `stock.quant.inventory_quantity`, `stock.quant.available_stock` | `stock`, `availablestock` | ตอน update product จะ update quant ตาม stock และ available stock ของ warehouse |
| Create/Update API lookup | `product.template.zort_id` | `id` | API create กัน duplicate ด้วย `zort_id`; API update หา product ด้วย `zort_id` |
| Delete API lookup | `product.template.sku` | `sku` | API delete หา product ด้วย `sku` แล้ว set `active = False` |

### Product Fields

| Fields | Odoo | Zort | Remake |
| ----- | ---- | ---- | ------ |
| zort_id | `product.template.zort_id` | `id` | เก็บ id จาก Zort ลง field `zort_id` ใน Odoo และใช้เป็น key sync |
| default_code | `product.template.default_code` | `sku` | เก็บ SKU จาก Zort เป็น Internal Reference |
| sku | `product.template.sku` | `sku` | มี custom field `sku`; delete endpoint ใช้ field นี้ตอนค้นหา |
| name | `product.template.name` | `name` | ชื่อสินค้า |
| description | `product.template.description` | `description` | รายละเอียดสินค้า |
| list_price | `product.template.list_price` | `sellprice` | ราคาขายจาก Zort |
| standard_price | `product.product.standard_price` / optional vals | `purchaseprice` | Batch sync ส่งเป็น optional `standard_price`; Create API เขียน purchase price ลง variant ถ้า `purchaseprice > 0` |
| barcode | `product.template.barcode` | `barcode` | barcode สินค้า |
| weight | `product.template.weight` | `weight` | น้ำหนัก |
| height | `product.template.height` | `height` | ความสูง |
| length | `product.template.length` | `length` | ความยาว |
| width | `product.template.width` | `width` | ความกว้าง |
| active | `product.template.active` | `active` | สถานะ active จาก Zort; delete API set เป็น `False` |
| type | `product.template.type` | - | ตั้งเป็น `consu` ตายตัว |
| is_storable | `product.template.is_storable` | - | ตั้งเป็น `True` ตายตัว |
| is_published | `product.template.is_published` | - | ตั้งเป็น `False` ตายตัว |
| categ_id | `product.template.categ_id` | `categoryid` | หา internal category จาก `product.category.zort_id`; ถ้าไม่เจอใช้ `product.product_category_all` |
| public_categ_ids | `product.template.public_categ_ids` | `categoryid`, `category` | หา/สร้าง public category จาก `categoryid` และ `category` แล้ว set ให้สินค้า |

### Category Mapping

| Fields | Odoo | Zort | Remake |
| ----- | ---- | ---- | ------ |
| public category lookup | `product.public.category.zort_id` | `categoryid` | หา public category ด้วย category id จาก Zort |
| public category create | `product.public.category.name`, `zort_id` | `category`, `categoryid` | ถ้าไม่เจอ public category และมีชื่อ category จะสร้างใหม่ |
| internal category lookup | `product.category.zort_id` | `categoryid` | ใช้ `_get_product_categ()` หา internal category เพื่อใส่ `categ_id` |
| internal category create | `product.category.name`, `zort_id`, `parent_id` | `category`, `categoryid` | ถ้ามี category แต่ยังไม่มี internal category จะสร้างใหม่ โดย `parent_id = False` |

### Stock Mapping

| Fields | Odoo | Zort | Remake |
| ----- | ---- | ---- | ------ |
| warehouse | `stock.warehouse.code` | `warehousecode` | ส่ง warehouse code ไป Zort ตอนดึง product list |
| stock quantity | `stock.quant.inventory_quantity` | `stock` | ใช้ stock จาก Zort เป็นจำนวน inventory แล้ว `_apply_inventory()` |
| available stock | `stock.quant.available_stock` | `availablestock` | เก็บ available stock จาก Zort ลง custom field บน quant |
| product available stock | `product.template.available_stock` | `stock.quant.available_stock` | compute รวม available stock จาก quant ใน location ของ warehouse บริษัท |
| product/location lookup | `stock.quant.product_id`, `stock.quant.location_id` | product + warehouse | หา quant จาก product variant และ warehouse stock location |

### Field ที่รับเข้ามาใน DTO แต่ยังไม่ถูกนำไปใช้โดยตรงใน Product Sync

| Fields | Odoo | Zort | Remake |
| ----- | ---- | ---- | ------ |
| producttype | - | `producttype` | DTO รับไว้ แต่ `_prepare_product_variant()` ไม่ map โดยตรง |
| sell_vat_status | - | `sell_vat_status` | DTO รับไว้ แต่ยังไม่ map tax ขาย |
| purchase_vat_status | - | `purchase_vat_status` | DTO รับไว้ แต่ยังไม่ map tax ซื้อ |
| unittext | - | `unittext` | DTO รับไว้ แต่ยังไม่ map UoM |
| image fields | - | `imagepath`, `imageList` | DTO รับไว้ แต่ยังไม่ download/set image |
| variation | - | `variationid`, `variant` | DTO รับไว้ แต่ยังไม่ map variant ใน Odoo |
| tag | - | `tag` | DTO รับไว้ แต่ยังไม่ map product tag |
| sharelink | - | `sharelink` | DTO รับไว้ แต่ยังไม่ map |
| properties | - | `properties` | DTO รับไว้ แต่ยังไม่ map |

## Contact Sync

ที่มา: `addons/tifshop_contact_sync`

- Batch sync ใช้ `Contact/GetContacts` ผ่าน `res.config.settings.action_sync_contacts()`
- ปุ่ม sync ใช้ `sync_contacts()` แล้วรัน background thread
- Logic mapping หลักอยู่ที่ `_prepare_partner_values()`

### Flow หลัก

| Step | Odoo | Zort | Remake |
| ----- | ---- | ---- | ------ |
| Get contacts | `_fetch_zort_contacts()` | `Contact/GetContacts` | ดึง contact จาก Zort โดยส่ง `page`, `limit` |
| Check response | `_fetch_zort_contacts()` | `res.resCode` | ต้องเป็น `200` ถ้าไม่ใช่จะ raise `UserError` |
| Load existing by Zort | `res.partner.zort_id` | `id` | สร้าง map `zort_id -> partner id` เพื่อ update contact เดิม |
| Load existing by name/ref | `res.partner.name`, `res.partner.ref` | `name`, `code` | สร้าง map `(name lower, ref) -> partner id` เป็น fallback |
| Skip invalid contact | - | missing `id` | ถ้า contact ไม่มี `id` จะข้าม |
| Update by Zort ID | `res.partner.write(vals)` | `id` | ถ้า `zort_id` ตรงกัน จะ update partner เดิม |
| Update by name/ref | `res.partner.write({**vals, "zort_id": id})` | `name`, `code`, `id` | ถ้าไม่เจอ zort_id แต่ชื่อ+ref ตรง จะ update partner เดิมและเติม `zort_id` |
| Create partner | `res.partner.create(vals)` | payload contact | ถ้าไม่เจอทั้งสองแบบ จะสร้าง partner ใหม่ |

### Contact Fields

| Fields | Odoo | Zort | Remake |
| ----- | ---- | ---- | ------ |
| zort_id | `res.partner.zort_id` | `id` | เก็บ id จาก Zort ลง partner และใช้เป็น key หลัก |
| ref | `res.partner.ref` | `code` | เก็บรหัสลูกค้าจาก Zort |
| name | `res.partner.name` | `name` | ชื่อลูกค้า/บริษัท |
| company_type | `res.partner.company_type` | `type` | map `Individual` เป็น `person`, `Corporate` เป็น `company`, ค่าอื่นเป็น `False` |
| vat | `res.partner.vat` | `idnumber` | เลขประจำตัวผู้เสียภาษี/เลขบัตร |
| phone | `res.partner.phone` | `phone` | เบอร์โทร |
| mobile | `res.partner.mobile` | `mobilePhone` | เบอร์มือถือ |
| email | `res.partner.email` | `email` | อีเมล |
| street | `res.partner.street` | `address` | ที่อยู่ |
| branch_name | `res.partner.branch_name` | `branchname` | ชื่อสาขา |
| branch_no | `res.partner.branch_no` | `branchno` | เลขสาขา |
| facebook | `res.partner.facebook` | `facebook` | Facebook |
| line | `res.partner.line` | `line` | Line |
| instagram | `res.partner.instagram` | `instagram` | Instagram |
| gender | `res.partner.gender` | `gender` | map `Male/Female/LGBT/Undefined` เป็น selection ใน Odoo |
| birth_date | `res.partner.birth_date` | `birthDate` | parse ISO date เป็น `YYYY-MM-DD`; ถ้า invalid ใช้ `False` |
| shipping_name | `res.partner.shipping_name` | `shippingName` | ชื่อผู้รับ/ชื่อที่อยู่จัดส่ง |
| shipping_phone | `res.partner.shipping_phone` | `shippingPhone` | เบอร์โทรที่อยู่จัดส่ง |
| shipping_address | `res.partner.shipping_address` | `shippingAddress` | ที่อยู่จัดส่ง |

### Company Type Mapping

| Fields | Odoo | Zort | Remake |
| ----- | ---- | ---- | ------ |
| company_type | `person` | `Individual` | บุคคลธรรมดา |
| company_type | `company` | `Corporate` | บริษัท |
| company_type | `False` | `Undefined` หรือค่าว่าง | ไม่ระบุประเภท |

### Gender Mapping

| Fields | Odoo | Zort | Remake |
| ----- | ---- | ---- | ------ |
| gender | `male` | `Male` | เพศชาย |
| gender | `female` | `Female` | เพศหญิง |
| gender | `lgbt` | `LGBT` | LGBT |
| gender | `other` | `Undefined` หรือค่าอื่น | ไม่ระบุ/อื่น ๆ |

### Field ที่รับมาจาก Zort แต่ยังไม่ถูกนำไปใช้โดยตรงใน Contact Sync

| Fields | Odoo | Zort | Remake |
| ----- | ---- | ---- | ------ |
| description | - | `description` | มีใน response ตัวอย่าง แต่ `_prepare_partner_values()` ไม่ map |
| imagepath | - | `imagepath` | ยังไม่ download/set image partner |
| sub contacts | - | `subContactList` | ยังไม่สร้าง child contacts |
| shipping list | - | `shippingList` | ใช้เฉพาะ flat fields `shippingName`, `shippingPhone`, `shippingAddress`; ไม่ map list |
| tag | - | `tag` | ยังไม่ map tag |
| properties | - | `properties` | ยังไม่ map custom properties |
