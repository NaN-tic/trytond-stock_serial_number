============================
Stock Serial Number Scenario
============================

=============
General Setup
=============

Imports::

    >>> import datetime
    >>> from dateutil.relativedelta import relativedelta
    >>> from decimal import Decimal
    >>> from proteus import config, Model, Wizard
    >>> from trytond.tests.tools import activate_modules
    >>> from trytond.modules.company.tests.tools import create_company, \
    ...     get_company
    >>> today = datetime.date.today()

Install stock Module::

  >>> config = activate_modules('stock_serial_number')

Create company::

    >>> _ = create_company()
    >>> company = get_company()

Create customer::

    >>> Party = Model.get('party.party')
    >>> customer = Party(name='Customer')
    >>> customer.save()

Create product::

    >>> ProductUom = Model.get('product.uom')
    >>> ProductTemplate = Model.get('product.template')
    >>> Product = Model.get('product.product')
    >>> unit, = ProductUom.find([('name', '=', 'Unit')])
    >>> product = Product()
    >>> template = ProductTemplate()
    >>> template.name = 'Product'
    >>> template.default_uom = unit
    >>> template.type = 'goods'
    >>> template.list_price = Decimal('20')
    >>> template.serial_number = True
    >>> template.save()
    >>> product.template = template
    >>> product.cost_price = Decimal('8')
    >>> product.save()

Get stock locations::

    >>> Location = Model.get('stock.location')
    >>> warehouse_loc, = Location.find([('code', '=', 'WH')])
    >>> supplier_loc, = Location.find([('code', '=', 'SUP')])
    >>> customer_loc, = Location.find([('code', '=', 'CUS')])
    >>> output_loc, = Location.find([('code', '=', 'OUT')])
    >>> storage_loc, = Location.find([('code', '=', 'STO')])

Create Shipment Out::

    >>> ShipmentOut = Model.get('stock.shipment.out')
    >>> shipment_out = ShipmentOut()
    >>> shipment_out.planned_date = today
    >>> shipment_out.customer = customer
    >>> shipment_out.warehouse = warehouse_loc

Add a line of 10 quantities of same product::

    >>> StockMove = Model.get('stock.move')
    >>> move = StockMove()
    >>> shipment_out.outgoing_moves.append(move)
    >>> move.product = product
    >>> move.unit = unit
    >>> move.quantity = 10
    >>> move.from_location = output_loc
    >>> move.to_location = customer_loc
    >>> move.unit_price = Decimal('1')
    >>> move.currency = company.currency
    >>> shipment_out.save()

Split the line into lots from 1 to 10::

    >>> Lot = Model.get('stock.lot')
    >>> shipment_out.reload()
    >>> first_lot = '1'
    >>> last_lot = '10'
    >>> move, = shipment_out.outgoing_moves
    >>> split = Wizard('stock.move.split', models=[move])
    >>> split.form.quantity == 1
    True
    >>> split.form.product == move.product
    True
    >>> split.form.start_lot = first_lot
    >>> split.form.end_lot = last_lot
    >>> split.form.count = None
    >>> split.execute('split')
    >>> shipment_out.reload()
    >>> len(shipment_out.outgoing_moves)
    10
    >>> lots = Lot.find([('product', '=', product.id)])
    >>> len(lots)
    10

We are not allowed to make a move of more than ::

    >>> move = StockMove()
    >>> move.product = product
    >>> move.unit = unit
    >>> move.quantity = 10
    >>> move.from_location = output_loc
    >>> move.to_location = customer_loc
    >>> move.unit_price = Decimal('1')
    >>> move.currency = company.currency
    >>> move.click('do') # doctest: +IGNORE_EXCEPTION_DETAIL
    Traceback (most recent call last):
        ...
    UserError: ...
