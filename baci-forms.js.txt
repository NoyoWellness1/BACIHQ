/**
 * BACI form handler — bacihq.com
 *
 * Include once per page, after the form markup:
 *
 *   <script src="/baci-forms.js" defer></script>
 *
 * Mark a form with data-baci-form="<kind>" and it is wired automatically.
 * The eight kinds are: contact, careers, thoughts, enterprise-access,
 * technology-partner-access, licensing, affiliate, support.
 *
 * The script adds the honeypot and render timestamp itself, so no page needs
 * to carry anti-spam markup. Nothing it injects is visible, and nothing it
 * renders is white — status messages use the site's gold variables.
 */

(function () {
  "use strict";

  var API = window.BACI_FORM_ENDPOINT || "https://api.bacihq.com/forms";

  function h(tag, attrs, text) {
    var el = document.createElement(tag);
    for (var k in attrs) if (attrs.hasOwnProperty(k)) el.setAttribute(k, attrs[k]);
    if (text) el.textContent = text;
    return el;
  }

  function injectHidden(form) {
    // Honeypot. Off-screen rather than display:none — some bots skip hidden
    // fields but fill positioned ones.
    var trap = h("div", {
      style: "position:absolute;left:-9999px;top:-9999px;" +
             "width:1px;height:1px;overflow:hidden;",
      "aria-hidden": "true"
    });
    var trapInput = h("input", {
      type: "text", name: "_honeypot", tabindex: "-1",
      autocomplete: "off", value: ""
    });
    trap.appendChild(trapInput);
    form.appendChild(trap);

    form.appendChild(h("input", {
      type: "hidden", name: "_renderedAt", value: String(Date.now())
    }));
    form.appendChild(h("input", {
      type: "hidden", name: "_sourcePage",
      value: window.location.pathname
    }));
  }

  function statusElement(form) {
    var existing = form.querySelector("[data-baci-form-status]");
    if (existing) return existing;

    var el = h("p", { "data-baci-form-status": "", role: "status" });
    el.style.cssText =
      "margin-top:1.25rem;font-family:'Cormorant Garamond',serif;" +
      "font-size:1.05rem;line-height:1.5;color:var(--gold,#C39544);" +
      "min-height:1.5em;";
    form.appendChild(el);
    return el;
  }

  function setStatus(el, message, tone) {
    el.textContent = message;
    el.style.color = tone === "error"
      ? "var(--gold-light,#CFA24E)"
      : "var(--champagne,#E8C87A)";
  }

  function fieldLabel(form, name) {
    var input = form.querySelector('[name="' + name + '"]');
    if (!input) return name;
    var id = input.getAttribute("id");
    if (id) {
      var label = form.querySelector('label[for="' + id + '"]');
      if (label) return label.textContent.replace(/[:*]\s*$/, "").trim();
    }
    return name;
  }

  function wire(form) {
    var kind = form.getAttribute("data-baci-form");
    if (!kind) return;

    injectHidden(form);
    var status = statusElement(form);
    var submitButton = form.querySelector('[type="submit"]');
    var originalLabel = submitButton ? submitButton.textContent : null;

    form.addEventListener("submit", function (event) {
      event.preventDefault();
      if (form.getAttribute("data-baci-submitting") === "true") return;

      form.setAttribute("data-baci-submitting", "true");
      if (submitButton) {
        submitButton.disabled = true;
        submitButton.textContent = "SENDING";
      }
      setStatus(status, "", "info");

      fetch(API + "/" + kind, {
        method: "POST",
        body: new FormData(form)
      })
        .then(function (response) {
          return response.json().then(function (data) {
            return { ok: response.ok, status: response.status, data: data };
          });
        })
        .then(function (result) {
          if (result.ok && result.data.ok) {
            // The form is replaced rather than reset. A cleared form invites
            // a second submission of the same thing.
            var done = h("div", { "data-baci-form-complete": "" });
            done.style.cssText =
              "font-family:'Cormorant Garamond',serif;font-size:1.15rem;" +
              "line-height:1.6;color:var(--champagne,#E8C87A);" +
              "padding:2rem 0;text-align:center;";
            done.textContent = completionMessage(kind);
            form.parentNode.replaceChild(done, form);
            return;
          }

          if (result.status === 400 && result.data.field) {
            setStatus(
              status,
              fieldLabel(form, result.data.field) + ": " + result.data.message,
              "error"
            );
            var input = form.querySelector('[name="' + result.data.field + '"]');
            if (input && input.focus) input.focus();
          } else {
            setStatus(
              status,
              result.data.message ||
                "Your submission could not be sent. Please try again.",
              "error"
            );
          }
        })
        .catch(function () {
          setStatus(
            status,
            "Your submission could not be sent. Please check your connection " +
            "and try again.",
            "error"
          );
        })
        .then(function () {
          form.removeAttribute("data-baci-submitting");
          if (submitButton) {
            submitButton.disabled = false;
            submitButton.textContent = originalLabel;
          }
        });
    });
  }

  function completionMessage(kind) {
    switch (kind) {
      case "careers":
        return "Your application has been received. You will hear from us "
             + "either way.";
      case "thoughts":
        return "Thank you. We have read it.";
      case "affiliate":
        return "Your application has been received. Approved affiliates "
             + "receive their tracking link and the programme terms on "
             + "approval.";
      case "licensing":
        return "Your enquiry has been received. We will be in touch to "
             + "arrange a conversation.";
      case "enterprise-access":
      case "technology-partner-access":
        return "Your request has been received. We will be in touch shortly.";
      case "support":
        return "Your request has been received. Response times are set out on "
             + "this page and vary by access route.";
      default:
        return "Your message has been received. We will respond within two "
             + "business days.";
    }
  }

  function init() {
    var forms = document.querySelectorAll("[data-baci-form]");
    for (var i = 0; i < forms.length; i++) wire(forms[i]);
  }

  if (document.readyState === "loading") {
    document.addEventListener("DOMContentLoaded", init);
  } else {
    init();
  }
})();
