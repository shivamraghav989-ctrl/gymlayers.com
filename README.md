// Mobile menu toggle and quick view modal
document.addEventListener('DOMContentLoaded', function(){
  const menuToggle = document.getElementById('menuToggle');
  const siteNav = document.getElementById('siteNav');
  menuToggle.addEventListener('click', ()=>{
    const expanded = menuToggle.getAttribute('aria-expanded') === 'true';
    menuToggle.setAttribute('aria-expanded', String(!expanded));
    // Toggle an "open" class; CSS handles the visible state for small screens
    siteNav.classList.toggle('open', !expanded);
  });

  // Quick view
  const modal = document.getElementById('modal');
  const modalTitle = document.getElementById('modalTitle');
  const modalPrice = document.getElementById('modalPrice');
  const modalClose = document.getElementById('modalClose');
  function openModal(name, price){
    modalTitle.textContent = name;
    modalPrice.textContent = price;
    modal.setAttribute('aria-hidden','false');
    document.body.style.overflow = 'hidden';
  }
  function closeModal(){
    modal.setAttribute('aria-hidden','true');
    document.body.style.overflow = '';
  }

  // Render products from the global PRODUCTS variable (loaded from js/products.js)
  const productGrid = document.getElementById('productGrid');
  // Cart elements
  const cartToggle = document.getElementById('cartToggle');
  const cartCount = document.getElementById('cartCount');
  const cartDrawer = document.getElementById('cartDrawer');
  const cartClose = document.getElementById('cartClose');
  const cartItemsEl = document.getElementById('cartItems');
  const cartTotalEl = document.getElementById('cartTotal');

  // Cart state: map productId -> qty
  let CART = {};

  function loadCart(){
    try{
      const raw = localStorage.getItem('demo_cart_v1');
      CART = raw ? JSON.parse(raw) : {};
    }catch(e){ CART = {}; }
    updateCartUI();
  }
  function saveCart(){
    localStorage.setItem('demo_cart_v1', JSON.stringify(CART));
    updateCartUI();
  }

  function addToCart(id, qty=1){
    CART[id] = (CART[id] || 0) + qty;
    saveCart();
    // announce add
    const prod = (window.PRODUCTS||[]).find(p=>p.id===id);
    const count = Object.values(CART).reduce((s,n)=>s+(n||0),0);
    announce(`${prod?prod.name:'Item'} added to cart. ${count} item${count!==1?'s':''} in your cart.`);
  }
  function removeFromCart(id){
    delete CART[id];
    saveCart();
    const prod = (window.PRODUCTS||[]).find(p=>p.id===id);
    const count = Object.values(CART).reduce((s,n)=>s+(n||0),0);
    announce(`${prod?prod.name:'Item'} removed from cart. ${count} item${count!==1?'s':''} remaining.`);
  }
  function setQty(id, qty){
    if(qty <= 0) removeFromCart(id);
    else CART[id] = qty;
    saveCart();
    const prod = (window.PRODUCTS||[]).find(p=>p.id===id);
    const count = Object.values(CART).reduce((s,n)=>s+(n||0),0);
    announce(`Quantity updated for ${prod?prod.name:'item'}. ${qty} in cart. ${count} total item${count!==1?'s':''}.`);
  }

  // ARIA live announcements for assistive tech
  const cartLive = document.getElementById('cartLive');
  function announce(message){
    if(!cartLive) return;
    // Clear quickly to ensure repeated same messages are announced
    cartLive.textContent = '';
    setTimeout(()=>{ cartLive.textContent = message; }, 50);
  }

  // Focus trap helpers for the cart drawer (accessibility)
  let _previousFocus = null;
  const _focusableSelector = 'a[href], area[href], input:not([disabled]), select:not([disabled]), textarea:not([disabled]), button:not([disabled]), [tabindex]:not([tabindex="-1"])';
  let _focusable = [];
  let _keydownHandler = null;

  function trapFocus(container){
    if(!container) return;
    _previousFocus = document.activeElement;
    container.setAttribute('aria-hidden','false');
    document.body.style.overflow = 'hidden';
    _focusable = Array.from(container.querySelectorAll(_focusableSelector)).filter(el => el.offsetParent !== null || el.tabIndex >= 0);
    // focus the close button if present, otherwise the first focusable element
    const preferred = container.querySelector('#cartClose') || _focusable[0];
    if(preferred) preferred.focus(); else container.focus();

    _keydownHandler = function(e){
      if(e.key === 'Escape'){
        e.preventDefault();
        closeCart();
        return;
      }
      if(e.key === 'Tab'){
        if(_focusable.length === 0){
          e.preventDefault();
          return;
        }
        const first = _focusable[0];
        const last = _focusable[_focusable.length - 1];
        if(e.shiftKey){
          if(document.activeElement === first){
            e.preventDefault();
            last.focus();
          }
        } else {
          if(document.activeElement === last){
            e.preventDefault();
            first.focus();
          }
        }
      }
    };
    document.addEventListener('keydown', _keydownHandler);
  }

  function releaseFocus(container){
    if(!container) return;
    container.setAttribute('aria-hidden','true');
    document.body.style.overflow = '';
    if(_keydownHandler) document.removeEventListener('keydown', _keydownHandler);
    _keydownHandler = null;
    // restore focus to the previously focused element
    try{ if(_previousFocus && typeof _previousFocus.focus === 'function') _previousFocus.focus(); }catch(e){}
    _previousFocus = null;
    _focusable = [];
  }

  function openCart(){ trapFocus(cartDrawer); }
  function closeCart(){ releaseFocus(cartDrawer); }

  cartToggle && cartToggle.addEventListener('click', ()=>{ const expanded = cartToggle.getAttribute('aria-expanded')==='true'; cartToggle.setAttribute('aria-expanded', String(!expanded)); openCart(); });
  cartClose && cartClose.addEventListener('click', closeCart);
  document.getElementById('checkoutBtn') && document.getElementById('checkoutBtn').addEventListener('click', ()=>{ alert('Checkout demo — no real payment.'); });

  function renderProducts(){
    const list = window.PRODUCTS || [];
    productGrid.innerHTML = '';
    list.forEach(p => {
      const article = document.createElement('article');
      article.className = 'product-card';
      article.dataset.id = p.id;
      article.dataset.name = p.name;
      article.dataset.price = p.price;
      article.dataset.priceNum = p.priceNum || (parseFloat((p.price||'').replace(/[^0-9.]/g,'')) || 0);
      article.innerHTML = `
        <div class="product-image"><img src="${p.image}" alt="${p.name}" class="product-img"></div>
        <div class="product-info">
          <h3>${p.name}</h3>
          <p class="price">${p.price}</p>
          <div class="actions">
            <button class="btn outline quickview">Quick view</button>
            <button class="btn addcart">Add to cart</button>
          </div>
        </div>`;
      productGrid.appendChild(article);

      // Enhance image loading: lazy, async decoding, and responsive srcset/sizes
      const img = article.querySelector('img.product-img');
      if(img){
        try{
          img.loading = 'lazy';
          img.decoding = 'async';
        }catch(e){}
        // If a `srcsetData` is present (embedded demo data-URIs), prefer that.
        if(p.srcsetData){
          img.srcset = p.srcsetData;
          img.sizes = `(max-width:900px) 100vw, (max-width:1200px) 50vw, 25vw`;
          img.src = p.srcFallback || p.image;
        } else if(p.srcsetBase){
          // expects files like p1-400.jpg, p1-800.jpg, p1-1200.jpg and webp variants
          img.srcset = `${p.srcsetBase}-400.webp 400w, ${p.srcsetBase}-400.jpg 400w, ${p.srcsetBase}-800.webp 800w, ${p.srcsetBase}-800.jpg 800w, ${p.srcsetBase}-1200.webp 1200w, ${p.srcsetBase}-1200.jpg 1200w`;
          img.sizes = `(max-width:900px) 100vw, (max-width:1200px) 50vw, 25vw`;
          // set a fallback src to a mid-sized jpg if available
          img.src = `${p.srcsetBase}-800.jpg`;
        } else {
          // Fallback demo: use the SVG placeholders (no real srcset benefit)
          img.srcset = `${p.image} 400w, ${p.image} 800w, ${p.image} 1200w`;
          img.sizes = `(max-width:900px) 100vw, (max-width:1200px) 50vw, 25vw`;
        }
      }
    });
    attachProductListeners();
  }

  function attachProductListeners(){
    document.querySelectorAll('.quickview').forEach(btn=>{
      btn.addEventListener('click', (e)=>{
        const card = e.target.closest('.product-card');
        const name = card.dataset.name || 'Product';
        const price = card.dataset.price || '';
        openModal(name, price);
      });
    });
    document.querySelectorAll('.addcart').forEach(btn=>{
      btn.addEventListener('click', (e)=>{
        const card = e.target.closest('.product-card');
        const id = card.dataset.id;
        addToCart(id, 1);
        // small feedback
        btn.textContent = 'Added';
        setTimeout(()=> btn.textContent = 'Add to cart', 900);
      });
    });
  }

  function updateCartUI(){
    // update badge
    const totalItems = Object.values(CART).reduce((s,n)=>s+(n||0),0);
    cartCount.textContent = totalItems;

    // render cart items
    const productsById = (window.PRODUCTS||[]).reduce((acc,p)=>{ acc[p.id]=p; return acc; },{});
    cartItemsEl.innerHTML = '';
    let total = 0;
    if(totalItems===0){
      cartItemsEl.innerHTML = '<p class="muted">Your cart is empty.</p>';
    } else {
      Object.keys(CART).forEach(id=>{
        const qty = CART[id];
        const p = productsById[id];
        if(!p) return;
        const row = document.createElement('div');
        row.className = 'cart-item';
        const itemTotal = (p.priceNum || parseFloat((p.price||'').replace(/[^0-9.]/g,''))||0) * qty;
        total += itemTotal;
        row.innerHTML = `
          <img src="${p.image}" alt="${p.name}">
          <div class="meta">
            <div class="name">${p.name}</div>
            <div class="qty">Quantity: <input type="number" min="0" value="${qty}" data-id="${id}" class="cart-qty" style="width:64px"></div>
            <div class="price">${p.price} · <strong>$${itemTotal.toFixed(2)}</strong></div>
          </div>
          <div><button class="btn outline remove" data-id="${id}">Remove</button></div>`;
        cartItemsEl.appendChild(row);
      });
    }
    cartTotalEl.textContent = `$${total.toFixed(2)}`;

    // wire remove/qty handlers
    cartItemsEl.querySelectorAll('.remove').forEach(b=> b.addEventListener('click', (e)=>{ removeFromCart(e.target.dataset.id); }));
    cartItemsEl.querySelectorAll('.cart-qty').forEach(input=> input.addEventListener('change', (e)=>{ const id = e.target.dataset.id; const val = parseInt(e.target.value||0,10); setQty(id, isNaN(val)?0:val); }));
  }

  // Kick off rendering
  renderProducts();

  // load cart state
  loadCart();

  modalClose.addEventListener('click', closeModal);
  modal.addEventListener('click', (e)=>{ if(e.target===modal) closeModal(); });

  // Accessibility: close modal with Esc
  document.addEventListener('keydown', (e)=>{ if(e.key==='Escape') closeModal(); });
});
