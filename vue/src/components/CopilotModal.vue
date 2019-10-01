<template>
  <Gridlayout rows="*" columns="*" ref="container" v-if="computedCopilotVisible" @loaded="onLoaded()" >
    <ViewMask row="0" height="100%"
      :animated="computedCurrentStep.animated"
      :size="computedSize"
      :position="computedPosition"
      :layout="computedLayout"
      :easing="easing"
      :animationDuration="animationDuration"
      :overlayColor="overlayColor"
    />
    <StepNumber row="0" height="100%"
      :animated="computedCurrentStep.animated"
      :size="computedSize"
      :position="computedPosition"
      :layout="computedLayout"
      :easing="easing"
      :animationDuration="`100`"
      :accentColor="accentColor"
      :backgroundColor="backgroundColor"
      :stepNumber="computedCurrentStep.order"
      :stepNumberPosition="computedStepNumberPosition"
      :safeArea="computedSafeArea"
    />
    <Tooltip row="0" ref="tooltip" 
      :currentStep="computedCurrentStep"
      :handleNext="next"
      :handlePrev="prev"
      :handleStop="stop"
      :labels="labels"
      :tooltipStyle="tooltipStyle"
      :tooltipPosition="computedTooltip"
      :layout="computedLayout"
      :tooltipMargin="margin"
      :arrowClipPath="computedArrowClipPath"
      :arrow="computedArrow"
      padding="0"
    />   
  </Gridlayout>  
</template>
<script lang="ts">
  import { Component, Vue, Prop } from 'vue-property-decorator';
  import * as platform from "tns-core-modules/platform";
  import * as app from 'tns-core-modules/application';

  import ViewMask from './ViewMask.vue';
  import Tooltip from './Tooltip.vue';
  import StepNumber from './StepNumber.vue';

  import { Step, Layout, ValueXY, ArrowPosition, TooltipPosition, Points, ViewLocationProperties } from '../utils/types';  
  import { MARGIN, ARROW_SIZE, OFFSET_WIDTH, ANDROID_STATUSBAR_OFFSET } from '../utils/constants';

  @Component({
    name: 'copilot-modal',
    components: {
      ViewMask,
      Tooltip,
      StepNumber
    }
  })
  export default class CopilotModal extends Vue {

    private tooltip: TooltipPosition = {
      left: 0,
      top: 0, 
      right: 0, 
      bottom: 0,
      middle: '*',
      alignment: 'left'
    };

    private arrow: ArrowPosition = {
      width: 0,
      height: 0,
      left: 0,
      top: 0, 
      right: 0, 
      bottom: 0,
      location: ''
    };

    private arrowClipPath: string = '';

    private animatedValues!: object;
    private notAnimated!: boolean;
    private layout: Layout = {
      width: 0,
      height: 0
    }; 

    private stepNumberPosition: Points = {
      left: 0,
      top: 0, 
      right: 0, 
      bottom: 0
    };  

    private safeArea: Points = {
      left: 0,
      top: 0, 
      right: 0, 
      bottom: 0
    };  
    private size: ValueXY = { 
      x: 0,
      y: 0
    };
    private position: ValueXY  = { 
      x: 0,
      y: 0
    };

    private margin = MARGIN;
    private screenScale!: number;

    private currentStep: Step = {
      name: 'First',
      text: 'here is some text',
      order: 1,
      target: '',
      animated: true
    };
    private stepCount: number = 0;
    private copilotVisible: boolean = false;

    private loaded: boolean = false;

    @Prop() public steps!: Step[];
    @Prop({ default: 300 }) public animationDuration!: number;
    @Prop({ default: {
      fontFamily: 'Avenir-Light',
      tooltipFontSize: 14,
      tooltipTextColor: 'black',
      buttonFontSize: 14,
      accentColor: 'green'
    }}) public tooltipStyle!: object;  
    @Prop({ default: false }) public androidStatusBarVisible!: boolean;
    @Prop({ default: 'rgba(0, 0, 0, 0.4)'}) public overlayColor!: string;
    @Prop({default: {
      skip: 'Skip',
      previous: 'Previous',
      next: 'Next',
      finish: 'Finish'
    }}) public labels!: object;
    @Prop({default: 'green'}) public accentColor!: string;
    @Prop({default: 'white'}) public backgroundColor!: string;

    // @Prop() public tooltipComponent!: ?React$Component;
    // @Prop() public stepNumberComponent!: ?React$Component;
    // @Prop() public easing!: Function;

    private onLoaded(): void {
      if (platform.isAndroid) {
        this.getDeviceInfoAndroid();
      } else {
        this.getDeviceInfoIOS();
      }

      // @ts-ignore
      const page = this.$refs.container.nativeView;
      let safeArea: Points = page.getSafeAreaInsets();

      if(platform.isAndroid) {
        safeArea = {
          left: safeArea.left,
          top: safeArea.top === 0 ? ANDROID_STATUSBAR_OFFSET : safeArea.top,
          right: safeArea.right,
          bottom: safeArea.bottom
        } 
      } else {
        safeArea = {
          left: safeArea.left / this.screenScale,
          top: safeArea.top / this.screenScale,
          right: safeArea.right / this.screenScale,
          bottom: safeArea.bottom / this.screenScale
        }
      }
      this.safeArea = safeArea;
      this.loaded = true;
      this.currentStep = this.steps[0];

    }

    private getDeviceInfoAndroid(): void {
      // @ts-ignore
      const metrics: android.util.DisplayMetrics = app.android.context.getResources().getDisplayMetrics();
      this.screenScale = metrics.density;

      const config = app.android.context.getResources().getConfiguration();
      this.layout = {
        height: config.screenHeightDp,
        width: config.screenWidthDp
      }
    }

    private getDeviceInfoIOS(): void {
      // @ts-ignore
      const screen = UIScreen.mainScreen;
      this.layout = {
        height: screen.nativeBounds.size.height / screen.scale,
        width: screen.nativeBounds.size.width / screen.scale
      }

      this.screenScale = screen.scale;
    }

    private async animateMove(view: any, verticalOffset: number): Promise<void> {
      // console.log('staring animateMove')
      
      const layout = {...this.layout};
      const tooltip: TooltipPosition = {...this.tooltip};
      const tooltipStyle: object =  {...this.tooltipStyle};
      const arrow: ArrowPosition =  {...this.arrow};
      let arrowClipPath: string =  '';

      const dim = view.getActualSize();
      const pos = view.getLocationInWindow();

      // console.log('obj - dim ', dim)
      // console.log('obj - - pos ', pos)

      const obj: ViewLocationProperties = {
        width: dim.width + OFFSET_WIDTH,
        height: dim.height + OFFSET_WIDTH,
        left: pos.x - (OFFSET_WIDTH / 2),
        top: ((pos.y - (platform.isAndroid ? this.safeArea.top : 0)) - (OFFSET_WIDTH / 2))  + verticalOffset 
      }

      const objCenter: ValueXY = {
        x: obj.left + (obj.width / 2),
        y: obj.top + (obj.height / 2),
      };

      const relativeToLeft = objCenter.x;
      const relativeToTop = objCenter.y;
      const relativeToBottom = Math.abs(objCenter.y - layout.height);
      const relativeToRight = Math.abs(objCenter.x - layout.width);

      const verticalPosition = relativeToBottom > relativeToTop ? 'bottom' : 'top';
      const horizontalPosition = relativeToLeft > relativeToRight ? 'left' : 'right';

      // console.log('onAnimate - obj ', obj);
      // console.log('onAnimate - layout ', layout);

      const objRight = Math.max(layout.width - (obj.left + obj.width), 0) ;
      const objLeft =  Math.max(obj.left, 0);
      // console.log('objRight - ', objRight);
      // console.log('objLeft - ', objLeft);

      tooltip.top = "*"
      tooltip.bottom = "*"
      tooltip.left = "*"
      tooltip.right = "*"
      tooltip.middle = "auto";

      arrow.height = ARROW_SIZE;
      arrow.width = ARROW_SIZE * 2;
      arrow.right ='*';
      arrow.left = '*';

      if (verticalPosition === 'bottom') {
        // console.log('BOTTOM BOTTOM BOTTOM BOTTOM')
        tooltip.top = ((obj.top + obj.height) - (platform.isAndroid ? 0 : this.safeArea.top)) + MARGIN;

        arrow.location = 'top';
        arrowClipPath = `polygon(50% 0%, 0% 100%, 100% 100%)`;

      } else {
        // console.log('TOP TOP TOP TOP')
        tooltip.bottom = (layout.height - (obj.top + (this.safeArea.bottom - this.safeArea.top))) - MARGIN 

        arrow.location = 'bottom';
        arrowClipPath = `polygon(50% 100%, 0% 0%, 100% 0%)`;
      }

      if (horizontalPosition === 'left') {
        // console.log('LEFT LEFT LEFT LEFT');
        tooltip.right = objRight //< MARGIN ? objRight + MARGIN : objRight;
        //  tooltip.left = objLeft < MARGIN ? objLeft + MARGIN : objLeft;
        tooltip.alignment = 'right';
        arrow.right = objRight < MARGIN ? objRight + (MARGIN * 2) : objRight + MARGIN;
      } else {
        // console.log('RIGHT RIGHT RIGHT RIGHT');
        // tooltip.right = objRight < MARGIN ? objRight + MARGIN : objRight;
        tooltip.left = objLeft //< MARGIN ? objLeft + MARGIN : objLeft;
        tooltip.alignment = 'left';
        arrow.left = objLeft < MARGIN ? objLeft + (MARGIN * 2) : objLeft + MARGIN;
      }

      this.tooltip = tooltip;
      this.tooltipStyle = tooltipStyle;
      this.arrow = arrow;
      this.arrowClipPath = arrowClipPath;
      this.layout = layout;

      this.size = {
        x: obj.width,
        y: obj.height,
      };

      this.position = {
        x: (Math.max(obj.left, 0)),
        y: (Math.max(obj.top, 0)),
      };

    }

    public start(): void {
      this.copilotVisible = true;
    }

    private next(): void {
      this.stepCount = this.stepCount === this.steps.length - 1 ? 0 : this.stepCount + 1;
      this.currentStep = this.steps[this.stepCount];
    }

    private prev(): void {
      this.stepCount = this.stepCount ===  0 ? this.steps.length - 1 : this.stepCount - 1;
      this.currentStep = this.steps[this.stepCount];
    }

    private stop(): void {
      this.copilotVisible = false;
      this.stepCount = 0;
    }

    get computedCopilotVisible(): boolean {
      return this.copilotVisible;
    }

    get computedCurrentStep(): Step {
      const verticalOffset = this.currentStep.verticalOffset === undefined ? 0 : this.currentStep.verticalOffset
      if(this.currentStep.target && this.loaded) {
        this.animateMove(this.currentStep.target, verticalOffset); 
      }
      return this.currentStep;
    }

    get computedSize(): ValueXY {
      return this.size;
    }

    get computedPosition(): ValueXY {
      return this.position;
    }

    get computedLayout(): Layout {
      return this.layout;
    }

    get computedTooltip(): TooltipPosition {
      return this.tooltip;
    }

    get computedTooltipStyle(): object {
      return this.tooltipStyle;
    }

    get computedArrow(): ArrowPosition {
      return this.arrow;
    }

    get computedArrowClipPath(): string {
      return this.arrowClipPath;
    }

    get computedSafeArea(): Points {
      return this.safeArea;
    }

  }

</script>

<!-- Add "scoped" attribute to limit CSS to this component only -->
<style lang="scss">
  // @import '../src/components/style.scss';

</style>